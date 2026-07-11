# corne-mouse-zmk

ZMK config that turns a **Corne v2** into a keyboard whose **right half doubles
as a desk mouse** — an optical sensor mounted face-down under the right half,
so you pick that half up, slide it on the desk, and it drives the cursor. No
trackball, no trackpad. This is the [Scoot](https://github.com/) desk-mouse idea
applied to an off-the-shelf wireless Corne.

> Firmware only. The mechanical side (optical window, ~10 mm sensor standoff,
> low-friction feet under the right half) is on you.

## This build's setup

- **Board:** Corne v2, two nRF52840 "Pro Micro"/SuperMini clones (built as
  `nice_nano_v2` — see [caveat](#board-caveat)).
- **Power:** no battery. **Both halves plugged into USB for power.** The left
  half is the USB HID device to the PC; the right half's USB is power only.
- **Split link:** **BLE** between the halves (data), even though both are on USB
  power. Keys and pointer motion from the right half cross over Bluetooth.
- **Pointing sensor:** **PMW3360** on the right half, over SPI, driven by
  [`george-norton/zmk-driver-pmw3360`](https://github.com/george-norton/zmk-driver-pmw3360).
- **Interaction:** hold-to-mouse — hold the left thumb SPACE (`&lt 1`, the
  `upper` layer, which already carries `&mkp MB1/MB2`); slide the right half to
  move and click. Turned on by uncommenting `layers = <1>;` in `mouse.dtsi`.

> The keymap and BLE tuning here are copied verbatim from
> [`xonha/zmk-corne`](https://github.com/xonha/zmk-corne); the only addition is
> the sensor (`mouse.dtsi` + `corne_right.conf` + the driver in `west.yml`).

## Layout of this repo

| File | Purpose |
| --- | --- |
| `config/west.yml` | Pulls in ZMK + the PMW3360 driver module. |
| `config/corne.keymap` | Your zmk-corne keymap verbatim, **plus** `#include "mouse.dtsi"`. |
| `config/mouse.dtsi` | The sensor: SPI node, pins, input-split channel, per-side split. |
| `config/corne.conf` | Kconfig for **both** halves (pointing + BLE split). |
| `config/corne_right.conf` | Kconfig for the **right** half only (SPI + driver). |
| `build.yaml` | GitHub Actions build matrix (left, right, settings_reset). |

### How the pointer is split across the halves

ZMK only auto-loads `<name>.conf` and `<name>.keymap` from `config/`, so all the
devicetree lives in `corne.keymap` (via `#include "mouse.dtsi"`), which applies
to **both** halves. `mouse.dtsi` then uses `#if CONFIG_ZMK_SPLIT_ROLE_CENTRAL`:

- **Left (central):** enables the `zmk,input-listener` and speaks USB HID.
- **Right (peripheral):** defines the physical PMW3360 on SPI and forwards its
  motion to the central over the BLE split via a shared `zmk,input-split`.

## ⚠️ Before this can work — TODO checklist

This repo is a **correct scaffold**, not a flashable-as-is firmware. You must:

1. **Buy the sensor.** A PMW3360 (or PMW3389) breakout **with the lens already
   fitted** — AliExpress/Tindie. The fitted lens is what makes desk tracking
   work; do not buy a bare sensor.
2. **Wire it to the right half** and fill in the real pins in
   [`config/mouse.dtsi`](config/mouse.dtsi) — every `TODO`: `SCK`, `MOSI`,
   `MISO`, `CS`, and `MOTION` (irq). Pick 5 free GPIOs on the right nice!nano.
3. **Confirm the board target.** See [caveat](#board-caveat).
4. **First flash: verify raw tracking** with the mouse layer's `layers` gate
   *disabled* (default). Slide the right half — the cursor should move.
5. **Then enable hold-to-mouse:** uncomment `layers = <1>;` in `mouse.dtsi` so
   the cursor only moves while the `upper` layer (left thumb SPACE) is held.
6. **Fix orientation** if X/Y are swapped/inverted (a face-down sensor usually
   is): `rotate-90;` and/or the `input-processors` line in `mouse.dtsi`.

### Board caveat

nRF52840 "Pro Micro"/SuperMini clones are *usually* pin-compatible with the
nice!nano v2, so `build.yaml` targets `nice_nano_v2`. Confirm **your** clone's
bootloader accepts a `nice_nano_v2` UF2 before trusting it — some need their own
board definition. If yours differs, tell me the exact model and I'll adjust the
board target and the SPI pin map.

## Building & flashing

**GitHub Actions (recommended):** push this repo to GitHub; the ZMK build action
produces `corne_left.uf2`, `corne_right.uf2`, and `settings_reset.uf2` as
artifacts. Put each half in bootloader (double-tap reset) and drag the matching
UF2 onto the USB drive that appears.

**Re-pairing:** if the halves won't talk after reflashing, flash
`settings_reset.uf2` to **both** halves, then flash the real firmware again.

**Local build (optional):**
```sh
west init -l config && west update
west build -s zmk/app -b nice_nano_v2 -- -DSHIELD=corne_right
```

## Notes & trade-offs

- **BLE latency on a mouse.** Data between halves is Bluetooth; for a pointer
  that's a real (if small) latency cost. If it bothers you, ZMK's wired UART
  split is the lower-latency path — but on a *desk-mouse* it adds a second
  trailing cable to the half you slide, so it was passed over here.
- **ZMK revision.** `west.yml` and the workflow are pinned to `v0.3` (matching
  zmk-corne). Peripheral pointing needs `zmk,input-split`; if the right-half
  build errors on those nodes, bump `zmk` in `west.yml` and the workflow ref to
  a newer tag together.
- **The sliding half still has one cable** (its USB power lead). It must flex
  and extend across your whole mousing range — a coiled/retractable lead or a
  generous service loop helps. This is the classic desk-mouse tether problem.
