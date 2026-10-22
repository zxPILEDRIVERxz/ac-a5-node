# AC Bus Node — wiring schematic (C6 chain, RX + gated TX)

Last updated: 2026-10-21 (FINAL WIRE MAP + UART moved to D5/D4). For: **XIAO ESP32C6** (Seed Studio)
+ bidirectional level shifter + USB breakout, tapping the mini-split's USB
extension cable.
Firmware in service: `ac-a5-control-c6.yaml` (session 2: listener + "TX Armed"
interlock + command entities). Full project context: `ac-uart-takeover.md`.

**Power plan:** test period = XIAO on a USB-C wall adapter; permanent install
= harness **VBUS → XIAO 5V pin**, USB cable removed forever, all later firmware
updates over OTA. The two power sources are NEVER connected at once (backfeed).

> **Board assumption:** "Seed Studio C6" = **XIAO ESP32C6** (21×17.8 mm, USB-C,
> 10-pin header). Pin map below is from Seeed's official wiki. If your board's
> silkscreen differs, stop and tell me before soldering.

> **⚠ FINAL WIRE MAP (metered by hand 2026-10-20 — AUTHORITATIVE).**
> The USB breakout's **PIN LABELS DO NOT MATCH THE CABLE CONDUCTORS**. Never
> trust the silkscreen; use this table (also at the top of
> `ac-a5-control-c6.yaml`):
>
> | conductor | real identity | goes to |
> |-----------|---------------|---------|
> | **black** | **+5 V (VBUS)** — the true power rail | shifter **VCC_B**, and XIAO **5V pin** (permanent install only) |
> | **white** | **GND** (an earlier "white ≈ 5 V" reading was actually **−5 V vs black**) | shifter **GND_A + GND_B** + XIAO **GND** |
> | **green** | **mainboard → module data** — carries the traffic, decode verified | shifter **B1 → A1 → XIAO D5** (RX, GPIO23) |
> | **red** | **module → mainboard data = our TX TARGET** — idles ≈5 V high with the stock dongle plugged in, which is what fooled the first measurement into calling it VBUS | shifter **B2 → A2 → XIAO D4** (TX, GPIO22) |
>
> 2026-10-21 update: XIAO-side terminations moved **D7→D5** and **D6→D4** for
> the permanent install. Both are plain GPIOs (I2C SDA/SCL by default — no
> strapping/boot function); D6/D7 were the UART0 console pair and stay free
> for serial logging. Conductor identities above unchanged (metered 2026-10-20).
>
> History: the first build ran with the "ground" net (black, really +5 V) as
> GND and true GND (white) on a B-side data pin — i.e. −5 V out-of-spec on the
> shifter's B side — and it still decoded because the XIAO island had floated
> +5 V above AC ground. It was never correct; everything was reconnected per
> this map 2026-10-20, and arming TX afterwards produced accepted ACKs in HA.

## The picture

```
AC MAINBOARD ◄──► Tuya dongle (session 1: may stay plugged in — the tap is passive)
        │
   USB EXTENSION CABLE
    black=+5V  white=GND  green=main->mod data (RX)  red=mod->main data (TX)
         <- REAL conductor identities, metered 2026-10-20. Breakout silkscreen labels are
            WRONG -- see the FINAL WIRE MAP banner above.
        │
        │  OPTION 1 (T-tap): nick jacket, separate the 4 conductors, solder a
        │                   short jumper to each. Cable stays fully intact.
        │  OPTION 2 (no cut): plug the cable's free USB-A end into the USB
        │                   breakout socket / Dreame Breakout PCB's USB-A
        │                   pass-through and use its 4 inner pins.
        ▼
 ┌─────────────────────────────────────┐
 │            USB BREAKOUT             │
 │   VBUS        D-       D+      GND  │
 └───┬────────────┬────────┬──────────┬──────────┐
     │            │        │          │          │
  VBUS         spare    ┌──┴───┐      │          │
 (red)           (green)│ B1   │      │          │
     │                 │      │      │          │
     ├────────────────►│SHIFT.├──────┼──────────┤
     │  VCC_B (high)   │      │ GND_B├──────────┤
     │                 │ A1   │      │          │
     │                 └──┬───┘ GND_A├──────────┘
     │                    │          │
     │                    ▼          │
 ┌───┴────────────────────────────────┴───────────────┐
 │                   XIAO ESP32C6                     │
 │   3V3 ──────► VCC_A (low)      D5 (RX, GPIO23) ◄── A1
 │   GND ◄────────────────────────────────────────────┘  (common GND net)
  │   5V  ◄── VBUS jumper (net 6): PERMANENT INSTALL ONLY —
  │        breakout end left open during the test period; closed one
  │        time after the USB-C cable is removed (see "Permanent power")
 │   USB-C ──► USB wall adapter   ← ONLY power source for the XIAO during the TEST PERIOD
 └─────────────────────────────────────────────────────┘
```

## Net list (what gets soldered to what)

| # | From | To | Note |
|---|------|----|------|
| 1 | **black conductor** = +5 V (VBUS) — metered identity, not the silkscreen "VBUS" pin | shifter **VCC_B** (high side) | makes the high side 5V-referenced |
| 2 | **white conductor** = GND | shifter **GND_B** AND **GND_A** AND **XIAO GND** | ONE common ground — required, all tied together. (On a raw TXS0108E there is a single GND pin.) An earlier "white ≈ 5 V" reading was actually −5 V vs black (which is +5 V) — white really is ground. |
| 3 | **green conductor** = mainboard→module data | shifter **B1** (high side data, channel 1) | the traffic line — decode verified on this wire 2026-10-20 (`Frames` climbing) |
| 4 | shifter **A1** (low side data) | **XIAO D5** (GPIO23, plain GPIO — moved here 2026-10-21, was D7/GPIO17) | the traffic signal into the XIAO |
| 5 | XIAO **3V3** | shifter **VCC_A** (low side) | sets the low side to 3.3V |
| 6 | **black conductor** = +5 V | **XIAO 5V pin** | **PERMANENT INSTALL ONLY.** A designed power input (Seeed pin map: "VBUS — Power Input/Output"). Build the jumper at assembly with its free end left open + heatshrunk; close it only AFTER unplugging USB-C. Never powered with USB-C attached — backfeed. |
| 7 | **red conductor** = module→mainboard data (**OUR TX TARGET**) | shifter **B2** (high side data, channel 2) | **SESSION 2 TX.** Identified by metering + the arming test (ACKs accepted on this wire). Idles ≈5 V high with the stock dongle plugged in — that is what made it look like a power rail. Tuya must be UNPLUGGED before we drive it (TX Armed gate). |
| 8 | shifter **A2** (low side data, channel 2) | **XIAO D4** (GPIO22, plain GPIO — moved here 2026-10-21, was D6/GPIO16) | session 2 transmit. Driven only while the "TX Armed" switch is ON; idle-high and electrically a bystander otherwise. |

That is the entire circuit. **Everything else stays unconnected:**

- **XIAO 5V pin:** nothing during the test period (USB-C powers it); at
  permanent install it receives net 6 and the USB-C cable is gone. The two
  power sources are NEVER connected at once — backfeed destroys boards/ports
  (ACiQ rule).
- **Green conductor:** this unit's traffic line — wired to shifter B1 as built (net 3, decode verified 2026-10-20).
- **Shifter channel 2:** session-2 TX (nets 7/8: red conductor → B2/A2 → XIAO D4). Channels 3–8 remain unconnected.
- **XIAO pins: only D5, D4, GND, 3V3 used.** Do not touch D0/D1/D2 (strapping
  pins), D6/D7 (the UART0 console pair — freed by the 2026-10-21 move; keep
  free for serial logging), D8–D10 (native USB / SPI), the JTAG pads
  (MTCK/MTDO/MTDI/MTMS = GPIO 6/7/5/4), BOOT (GPIO9), or the RF-switch pins
  (GPIO3/GPIO14).

Shifter silkscreen variants (follow YOUR board):
- **4-channel module** (classic blue board): two VCC pins, two GND pins,
  A0–A3 low bank, B0–B3 high bank. Use any one channel (here: 1).
- **Raw TXS0108E chip or 8-ch board**: `VCCA` = 3.3V side, `VCCB` = 5V side,
  ONE GND, `A0..A7` / `B0..B7` pairs. The "E" variant **auto-detects
  direction — there is no DIR pin to wire**. Use pair 0 or 1.

## XIAO ESP32C6 pin map (reference — Seeed wiki)

| Pad | GPIO | Us here | Why not |
|-----|------|---------|---------|
| D0 / A0 | 0 | — | strapping pin |
| D1 / A1 | 1 | — | keep free |
| D2 / A2 | 2 | — | **strapping** (SPI voltage select) |
| D3 | 21 | session 2 spare RX | free now |
| **D4** | **22 (SDIO_DATA2)** | ✅ **our TX** | plain GPIO; moved here 2026-10-21 from D6 — frees the UART0 console TX pin |
| **D5** | **23 (SDIO_DATA3)** | ✅ **our RX** | plain GPIO; moved here 2026-10-21 from D7 — frees the UART0 console RX pin |
| D6 | 16 (UART0 TX) | — | console TX — now free, keep free for serial logging |
| D7 | 17 (UART0 RX) | — | console RX — now free, keep free for serial logging |
| D8/D9/D10 | 19/20/18 | — | native USB / SPI |
| MTCK..MTMS | 6/7/5/4 | — | JTAG test pads |
| BOOT | 9 | — | button (download mode) |
| EN | chip reset | — | button |
| LED | 15 | — | yellow user LED (status only) |

## Soldering procedure (order matters)

0. **Solder the XIAO headers first** (shipped unpinned). 2×5 pin headers,
   careful on the small pads — don't bridge. Flash-test the bare XIAO via
   USB-C BEFORE any tap wiring (see "Flashing" below). A dead XIAO at the end
   of the job is the worst day.

1. **Meter the cable (step 0 from the project doc).** AC powered ON. Trust the
   meter over any silkscreen — on this cable the breakout's pin labels do NOT
   match the conductors (that mislabeling is how we ended up −5 V out-of-spec):
   - **black ↔ white ≈ 5 V, black positive** → black = +5 V (VBUS), white = GND.
     (An earlier "white ≈ 5 V" reading was actually −5 V vs black.)
   - **red ≈ 5 V vs white while the stock dongle is plugged in**: red is a data
     line idling high (module→mainboard direction = our TX target), NOT a power
     rail — that idle-high reading is what made it look like VBUS at first.
   - **green vs white:** logic levels, toggles with traffic — the mainboard→
     module data line (RX). Decode verified on this wire.
   These are the measured identities (see the FINAL WIRE MAP banner above);
   re-metering is only needed if the cable or unit changes.

2. **Make the tap.**
   - *Option 1 (T-tap):* nick the extension cable's jacket, separate the four
     conductors, twist a short tinned jumper onto each and solder (or WAGO
     221-413 if you'd rather not heat the cable). The cable stays fully
     functional — Tuya can keep working whenever you want it.
   - *Option 2 (no cut):* plug the cable's free USB-A end into the breakout /
     Dreame Breakout PCB's USB-A socket; its 4 inner pins ARE VBUS/D-/D+/GND
     (map by silkscreen or continuity to the socket).
   Then connect the four nets to the breakout pins: VBUS / D- / D+ / GND.
   Insulate every joint (heatshrink or vinyl tape) and meter that no two
   breakout pins are shorted.

3. **Solder the shifter** per its silkscreen (see variants above), using the
   measured conductor identities from the FINAL WIRE MAP: VCC_B ← black
   (+5 V), GND(s) ← white (GND), B1 ← green (mainboard→module data). Leave
   VCC_A and A1 free until the XIAO side is soldered.

4. **Solder the XIAO side:** A1 ← XIAO **D5** (RX) and A2 ← XIAO **D4** (TX, session 2), GND net ← XIAO **GND**,
   VCC_A ← XIAO **3V3**, plus the **permanent-power jumper (net 6)**: one end
   on the XIAO **5V pin**, other end tinned, left UNCONNECTED with heatshrink
   on it, routed clear of everything. It closes at conversion time — see
   "Permanent power" below. Nothing else on the XIAO.

5. **Insulate and mount.** Heatshrink/tape over every joint; check no solder
   wick bridges XIAO pins or shifter pads (the shifter has 1.27 mm pitch —
   check under light). Zip-tie the assembly to the extension cable. Keep the
   XIAO's antenna edge clear of metal — if you're mounting inside the cabinet
   and WiFi is marginal, zip-tie it so the XIAO pokes out of the unit.

6. **Final meter sweep before powering:**
   - breakout VBUS ↔ GND: no short (continuity).
   - XIAO 3V3 pin ↔ GND: no short.
   - each shifter A/B data pair: open circuit until both ends are soldered,
     then low resistance (it's a passthrough when powered).

## Flashing

XIAO ESP32C6 has **native USB** on the USB-C port — no bridge chip.
1. The in-service firmware is `ac-a5-control-c6.yaml` — real home WiFi already
   filled in (credentials live in `/config/secrets.yaml`, referenced as
   `!secret wifi_password`; never paste them into chat);
   `board:` = `seeed_xiao_esp32c6`, RX = D5, TX = D4. The session-1 listener
   source remains `ac-a5-listen-c6.yaml`.
2. HA → ESPHome add-on → Add New Device → pick the XIAO over USB (data cable!)
   → paste the config → Flash. If not detected: hold **BOOT** while plugging
   in, release after ~2 s.
3. After it joins WiFi, all entities + log appear in HA. The yellow LED
   (GPIO15) shows firmware activity.

For the test period the XIAO stays powered from a USB wall adapter (or laptop)
— same cable that flashed it. Don't unplug it mid-test; there is no battery.
This first flash is the **only** time the USB cable is strictly required:
`ota:` is already in the config, so every later update goes over WiFi — and
after conversion to harness power the cable is not around at all.

## Permanent power (harness install) — one-time conversion

The permanent install has **no cables**: the XIAO runs on the AC unit's own
VBUS, the same rail that powers the Tuya dongle through this cable today.
The 5V pin is a designed input (Seeed's pin map: "VBUS — Power Input/Output")
— it IS the board's VBUS rail, so feeding it an external ~5 V source is normal
use of the board.

Guard rails:
- Step 1 already measured **VBUS ≥ ~4.5 V**. If it read lower, tell me before
  closing net 6 — the 3V3 regulator needs headroom during WiFi transmit bursts.
- At conversion the Tuya dongle is unplugged (the whole point of this project),
  so the rail carries only our node + shifter.

Procedure:
1. Finish testing on USB power (frames verified, map looks right).
2. **Unplug the USB-C cable from the XIAO.** It is never plugged in again.
3. Close the one open joint — the VBUS jumper's free end onto the breakout
   **VBUS** pin (or the existing VBUS net) — insulate it.
4. The node boots from the harness within seconds. Verify in HA: reconnects,
   `RX Bytes` ticks, no reboot spam in the log.
5. Leave it for a few hours; confirm no brownout reboots (see troubleshooting).
   Done — no cable ever again; all future updates are OTA.

## Power rules (the only way this can hurt you)

| Rule | Why |
|------|-----|
| XIAO power = exactly ONE source at a time — USB-C (test period) or harness VBUS → 5V pin (permanent) | Both connected at once = two 5V sources fighting = backfeed, destroyed boards/ports (ACiQ rule). Unplug USB-C **before** closing net 6 — every single time. |
| Common GND everywhere (breakout/shifter/XIAO) | required for any signal to make sense |
| Shifter VCC_B from VBUS, VCC_A from XIAO 3V3 | that's what makes it translate 5V↔3.3V |
| No 5V anywhere on the XIAO except its own USB port | C6 GPIOs are 3.3V-only; overvoltage kills them |

## Test plan (after power-up, AC on)

1. HA: `RX Bytes` should start ticking within a minute (the mainboard talks
   autonomously — room temp ~every 60 s, even with Tuya unplugged).
2. Wait ~5 min → check `Frames`, `CRC ACiQ-hole OK`, `CRC Zokop-prefix OK`.
   One CRC counter should dominate: that's your unit's variant → session 2.
3. Press the IR remote (power, temp ±, mode, fan) and watch `Power`, `Mode`,
   `Setpoint (F param)` track it. Screenshot/log `Last Frame` +
   `Unmapped Fields` — that's the mapping material.
4. **Bonus with Tuya left plugged in:** toggle things in the Smart Life / Tuya
   app and watch `Last Command` — decoded `0A 0A` command frames appear, which
   is exactly what session 2 needs for the command map. (Passive tap: safe.)
5. **When testing is done → permanent install** (see "Permanent power"):
   unplug USB-C, close the VBUS jumper, verify reconnect + a few quiet hours.
   No cable ever again.

Troubleshooting table:
| Symptom | Meaning |
|---------|---------|
| RX Bytes flat, AC on | wrong wire / no common GND / shifter miswired — re-meter steps 1–6 |
| RX Bytes rising, Frames = 0, CRC Bad rising | baud or framing off → try `baud_rate: 9600` in the config |
| RX Bytes + Frames OK but all fields unmapped | protocol family right, map differs from ACiQ's — that's fine, the queue is the work |
| Watchdog reboots ×3 and gives up | bus may only talk with Tuya attached → plug Tuya back in (safe) and retest |
| Node drops off WiFi / reboots randomly after conversion to harness power | mainboard rail sagging under WiFi bursts → re-measure VBUS with the node attached (want ≥ ~4.5 V); the Tuya dongle must be unplugged — it shares that rail |
