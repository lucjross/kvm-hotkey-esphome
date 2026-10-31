# KVM control from Home Assistant

An ESP32-S2 or ESP32-S3 board plugged into a console USB port on a KVM switch,
which *is* a keyboard: it types the KVM's own hotkey sequences on command from
Home Assistant.

Most desktop KVMs have no network interface and no API — the only ways in are
the front panel, maybe an IR remote, and console-keyboard hotkeys. This takes the
hotkey route, because it needs no modification to the KVM and coexists with the
real keyboard.

## Supported KVMs

Each file in `kvms/` is one KVM's hotkey set, turned into Home Assistant
entities. Its header comment lists the hotkeys it uses.

| File | KVM | Status |
|---|---|---|
| `scroll-lock-generic.yaml` | Unbranded 4-port "ScrLk ScrLk" KVM | Tested |
| `aten-cs1924m.yaml` | ATEN CS1924M | From the manual, untested |
| `level1techs.yaml` | Level1Techs 4-port KVMs | From the official hotkey list, untested |

Pick one in `kvm.yaml`:

```yaml
packages:
  device: !include common/device.yaml
  kvm: !include kvms/aten-cs1924m.yaml
```

Every entity is write-only. Nothing reads back which port the KVM is on, so
entities show what was last commanded, not ground truth, and toggles drift if
anyone uses the front panel. Reflecting real state would need a separate signal
per PC.

## Will it work with my KVM?

It will if all of these are true:

- **The KVM has keyboard hotkeys.** Many USB-3 "KVM switches" just switch a USB
  hub between computers and never read the keyboard, so there is nothing to type
  at.
- **There's a free console port that listens for hotkeys.** Some KVMs only watch
  one dedicated keyboard port. If your real keyboard is in it, the board needs
  another port that also reads hotkeys.
- **The KVM accepts a plain boot-protocol keyboard.** Most do. A few insist on a
  combined keyboard and mouse, or on known vendor IDs.

The hotkeys themselves can be almost anything a person can type:

- Plain sequences: `ScrLk ScrLk 1`, `NumLk NumLk Enter`.
- Modifier double-taps: `Ctrl Ctrl 1`, `Right-Ctrl Right-Ctrl ←`.
- Held chords of up to four keys: `Ctrl+F12`, `Num Lock` held with `-` tapped,
  `Ctrl+Alt+Shift+1`.

The one shape it can't do is a key held across *several* later steps, e.g.
"hold Shift, tap Num Lock twice, release Shift".

## Adding a KVM

Copy the closest file in `kvms/`, rewrite the `keys:` lists from your KVM's
manual, and point `kvm.yaml` at it. Refer to the keyboard as `id: kbd`. If the
KVM drops keys, raise `key_hold` / `key_gap` in `kvm.yaml`.

If it works, a pull request adding the file is welcome. Say in its header which
model and firmware you tested it on.

## Layout

```
kvm.yaml                          the config you build: picks a board and a KVM
common/device.yaml                board, WiFi, API, OTA, and the USB keyboard
kvms/                             one hotkey set per KVM
components/usb_hid_keyboard/      the USB HID keyboard component
secrets.yaml.example              copy to secrets.yaml
```

`kvm.yaml` overrides the defaults in `common/device.yaml` through
`substitutions:` — `device_name`, `friendly_name`, `board`, `key_hold` and
`key_gap`.

Under the Home Assistant ESPHome add-on, copy `kvm.yaml` and the `common/`,
`kvms/` and `components/` directories into `/config/esphome/`. `!secret` then
resolves against the add-on's own `secrets.yaml`.

## Why there's a custom component

ESPHome cannot present a device as a USB HID keyboard on its own. Its `tinyusb`
component registers only a CDC class, and a HID class would have to supply
`tinyusb_config_t.descriptor.full_speed_config` — but that struct is a
`protected` member of a `final` class, so it can be neither reached nor
subclassed from outside. The one community PR for this
([esphome#3917](https://github.com/esphome/esphome/pull/3917)) is closed, still a
draft, stacked on two other unmerged PRs, and Arduino-only.

`components/usb_hid_keyboard/` therefore bypasses ESPHome's `tinyusb` component
entirely and calls `tinyusb_driver_install()` itself, supplying its own device,
configuration and HID report descriptors. It pulls in `espressif/esp_tinyusb` and
sets `CONFIG_TINYUSB_HID_COUNT=1`.

It advertises a **boot-protocol** keyboard (interface class 3, subclass 1,
protocol 1) with unprefixed 8-byte reports. That matters: many KVMs only parse
boot-protocol reports, and adding a HID report ID would break exactly the hosts
this exists for. Modifiers travel in the report's modifier byte, not as keycodes,
because that is the only place a boot-protocol parser looks for them.

Keys are sent from `loop()` — press, hold `key_hold`, release, idle `key_gap` —
so nothing blocks WiFi or the API while a sequence plays out. A
sequence queued while the host isn't ready is dropped after 5 s rather than
replayed at the next plug-in, so keys pressed while unplugged cannot switch the
KVM unexpectedly later.

### Component options

```yaml
usb_hid_keyboard:
  id: kbd
  usb_manufacturer_str: "ESPHome"     # default "ESPHome"
  usb_product_str: "KVM Control"      # default "ESPHome Keyboard"
  usb_vendor_id: 0x303A               # default 0x303A
  usb_product_id: 0x4010              # default 0x4010
  usb_serial_str: ""                  # default: the MAC address
  key_hold: 20ms                      # how long each key is held
  key_gap: 30ms                       # idle gap before the next key
```

Raise `key_hold` / `key_gap` if a KVM drops keys out of a hotkey sequence.

### Action

```yaml
- usb_hid_keyboard.send_keys:
    id: kbd
    keys: [SCROLL_LOCK, SCROLL_LOCK, 2]
```

`keys` is a list of steps, typed in order. A step is one of:

- **A key name**: `SCROLL_LOCK`, `NUM_LOCK`, `LEFT`, `ENTER`, `ESCAPE`, `MINUS`,
  `F1`... A single character or a bare digit works too.
- **A modifier**: `LEFT_CTRL`, `RIGHT_CTRL`, `LEFT_SHIFT`, `RIGHT_SHIFT`,
  `LEFT_ALT`, `RIGHT_ALT`, `LEFT_GUI`, `RIGHT_GUI`. `CTRL`, `SHIFT`, `ALT`,
  `GUI`, `WIN` and `CMD` mean the left one. On its own, a modifier is tapped,
  so `[RIGHT_CTRL, RIGHT_CTRL, 1]` is a double-tap of Right Ctrl, then 1.
- **A chord** of up to four keys joined with `+`, e.g. `CTRL+F12` or
  `NUM_LOCK+MINUS`. Its keys go down in the order written, `key_gap` apart, stay
  held for `key_hold`, then come up in reverse. That is "hold Num Lock, tap
  minus, release Num Lock" exactly.

A bare string that isn't a key name or a chord is split into characters, so
`keys: FLASH` means F, L, A, S, H.

`keys` is also templatable. A lambda returns `std::vector<uint32_t>` of HID usage
codes. `usb_hid_keyboard::chord(a, b, ...)` packs a chord, and
`usb_hid_keyboard::keys_from_string()` turns digits and letters into keys. See
the interval numbers in `kvms/`, which build a variable-length sequence at
runtime.

## Building

Nothing needs to be installed locally:

```bash
cp secrets.yaml.example secrets.yaml     # dummy values are fine to compile
docker run --rm -v "$PWD":/config ghcr.io/esphome/esphome config  kvm.yaml
docker run --rm -v "$PWD":/config ghcr.io/esphome/esphome compile kvm.yaml
```

## Flashing

First flash is over USB. Hold **BOOT**, tap **RESET**, release BOOT — the board
enumerates as `303a:0002` on `/dev/ttyACM0` — then:

```bash
docker run --rm --device=/dev/ttyACM0 -v "$PWD":/config \
  ghcr.io/esphome/esphome run kvm.yaml --device /dev/ttyACM0 --no-logs
```

Everything after that goes OTA. Note that once this firmware is running the board
is a keyboard, not a serial port, so reflashing over USB means the BOOT+RESET
dance again.

## Verifying against a PC

Useful before trusting it at the KVM, because it separates "the HID device is
wrong" from "the KVM doesn't like it".

```bash
lsusb -v -d 303a:4010 | grep -E 'bInterface(Class|SubClass|Protocol)'
```

must report class 3, subclass 1, protocol 1. Then watch what it actually types:

```bash
sudo evtest /dev/input/by-id/*KVM_Control*-event-kbd    # apt install evtest
```

and trigger entities from Home Assistant. Each hotkey should appear as its
expected run of key-down events. `evtest` does not grab the device, so the
keystrokes *also* reach whatever window has focus — put focus in a scratch editor
first, or the board will type into your terminal and toggle Scroll Lock.
