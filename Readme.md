# Pico Reflow Oven

My own version of a custom reflow oven, built around my former kitchen oven and whatever was in
the drawers. Hence the "frankenstein" style: this time I'm not sourcing components for a project,
I'm building one from spare parts (plus a few things I still had to order).

I also reused a daughter board from a previous HV project ([NanoFridgeThermostat](https://github.com/bebenlebricolo/NanoFridgeThermostat)),
since it was designed as a self-contained opto-isolated triac switching board. A revisited
schematic for this project lives in `Electronics/PicoReflowOven/`.

## What is it for?

Reflowing custom PCBs in a 1500 W / 230 V convection oven that already has a high-temperature fan
and 2 sets of heating elements (top + bottom) — forced convection keeps the heat distribution as
even as possible.

- **Main target:** low-temp paste **Sn42Bi58** (melts at 138 °C, peak 170–180 °C)
- **Secondary target:** standard **SAC305 / SnAgCu** (peak ~245–250 °C) — reachable by
  disconnecting the original thermal cutout, in "bench" mode only (see Safety)

The stock oven dial goes to ~230 °C but only ever reached ~190–200 °C; measured ramp is
~0.6–0.67 °C/s, and the real fight is the **overshoot at the peak**, not the climb. Full analysis:
`Docs/DesignDeepDive.md`.

## Philosophy

Reuse parts, and keep the oven revertible to its stock configuration with minimal modifications
(original front panel and controls stay; a 2-pin connector restores the stock wiring in ~30 s).
Based on the stock configuration alone, low-temp paste is already reachable — what was missing is
a live temperature reading, a way to switch the elements, and an alarm to open the door.

## Safety (summary — the full architecture is part of the design, don't bypass it)

This project switches mains AC (230 V). Never touch live components. **No fuse, no thermal fuse**
(a shorted triac draws the same current as the load — no fuse would see it), so bounding is:

1. **Autonomous mode (SnBi only, default):** the original thermal cutout stays in series
   (~210–220 °C, above the 180 °C SnBi peak, independent of the Pico and triac). It trips below
   the SAC peak — SnBi only.
2. **Bench mode (any SAC run or test > 250 °C, cutout disconnected):** operator present at all
   times, mechanical timer armed, logical cutoffs ~300 °C.
3. **Mechanical timer armed at every run.** After every bench session, physically reconnect the
   cutout before any SnBi run. **SAC = bench mode only.**
4. House rules: thermal checks unplugged at residual heat; no PVC/thermoplastics in the cavity
   gap; HV/LV ≥ 6 mm apart in the enclosure; sacrificial probe bolted through a drilled FR4 plate.

Details: `Docs/DesignDeepDive.md` §3.

## Architecture — 3 layers, one golden rule

**The control loop and all safety mechanisms live on the Pico — the PC never supervises the hot
chain.** A run in progress survives a USB disconnect; PC/Bluetooth links are passive observers
(telemetry, settings, launch). Link loss mid-run = the run continues.

```
┌─────────────────────────────────────────────┐
│  Layer 3 — Local HMI                        │
│  SSD1306 OLED 128×64 + rotary encoder       │
│  + 2 buttons (Edit / Back)                  │
├─────────────────────────────────────────────┤
│  Layer 2 — USB protocol (CDC /dev/ttyACM0)  │
│  Commands: PROFILE, START, ABORT, GAINS,    │
│  MODE MANUAL x% · Telemetry CSV 1 Hz        │
├─────────────────────────────────────────────┤
│  Layer 1 — Autonomous control + safety      │
│  State machine RAMP/SOAK/PEAK/COOL/ALARM    │
│  PID 1 Hz on board probe · slow PWM         │
│  Interlocks, logical cutoffs                │
└─────────────────────────────────────────────┘
```

Layer 1 runs standalone (no PC, no HMI needed). Hardware, pinout (reference: the KiCad schematic),
measurement and protocol specs: `Docs/DesignDeepDive.md` and `Firmware/Readme.md`.

## Profiles

**SnBi first — the daily driver:**

| Phase | Setpoint |
|-------|----------|
| Free ramp | ~0.6 °C/s up to ~145 °C |
| Soak | 145–150 °C, 60–90 s |
| Peak | 170–180 °C |
| TAL | 30–60 s above 138 °C |
| Cooling | Door open + convection fan |

**SAC305 — later, bench mode only** (values [DATASHEET]): free ramp ~1–2 °C/s target to ~150 °C,
soak 150–180 °C 60–120 s, peak 245–250 °C, TAL 30–60 s above 217 °C, same cooling.

The profile table is extensible (paste alloy × board thermal mass) — not a hard-coded list.

## Using it

<<<<<<< HEAD
- **From the box:** OLED + encoder + Edit/Back. Edit = validate/START, Back = cancel; once
  running, the field becomes STOP and either button stops immediately.
- **From a Linux PC:** `/dev/ttyACM0` (native CDC, no reset on open — capture can start mid-cycle).
  Line protocol: `PROFILE <name>`, `START`, `ABORT`, `GAINS kp ki kd`, `MODE MANUAL x%`;
  1 Hz CSV telemetry `t,sp,tc_board,tc_air,duty,STATE` (e.g. `cat /dev/ttyACM0 | tee run.csv`).
  Group `dialout`; disable ModemManager. Nothing is logged before the port is opened. Spec:
  `Firmware/Readme.md`.
=======
---

## Safety — the part I won't bend on

**No fuse, no thermal fuse.** A triac shorted closed draws the same ~6.5 A as the load — no fuse would even notice that fault. So the bounding is done like this:

1. **Autonomous mode (SnBi only, default):** the oven's original thermal cutout goes back in series (~210–220 °C resettable, above the 180 °C SnBi peak, independent of the Pico and the triac). SnBi only — it trips below the SAC peak and can't protect a SAC run.
2. **Bench mode (any SAC run or test > 250 °C, cutout disconnected):** operator present at all times, mechanical timer armed, logical cutoffs ~300 °C.
3. **Mechanical timer armed at every run** — switch AND time bound. Any runaway dies with the dial.
4. **After every bench/SAC session: physically reconnect the cutout** before any SnBi run.
5. **SAC = bench mode only** (decided Nov 2026, to be logged as DEC-XXX). And no, I'm not shifting the cutout's trip point by blowing fresh air on it — that defeats the protection. Idea considered, **rejected**.

House rules:

- Thermal checks only with the plug out of the wall, at residual heat.
- Nothing 3D-printed in the cavity (air hits 170–180 °C on SnBi, up to ~250 °C on SAC — inside is FR4 and metal only).
- **No PVC in the cavity gap** (> 100 °C near the wall).
- HV/LV separated by ≥ 6 mm in the enclosure; airflow through the heatsink (intake low, exhaust high).
- The sacrificial probe is **bolted through a drilled FR4 plate** — mechanical fixing, no Kapton limit on the probe itself, SAC-compatible. Kapton is only for holding it in place on the target board.

---

## Hardware — mostly from the drawers

| Block | Detail |
|-------|--------|
| MCU | Raspberry Pi Pico (RP2040), C++ / pico-sdk — chosen over a Nano because the Nano's DTR reset on serial-open would kill a running cycle; Pico also adds UART1 (HC-05) and 264 KB RAM for the OLED |
| Power switching | BTA316Y-800CTQ triac (16 A / 800 V, 3Q, IGT 35 mA, factory-isolated tab) driven by MOC3042M opto-triac via NPN low-side |
| Daughter board | Reused from my "NanoFridgeThermostat" project (MOC3042M + snubber R 360 Ω / C 10 nF, TE 5.08 mm terminals, J5/J6 headers) |
| Temperature | 2× K-type probes + 2× MAX6675 (shared SPI0): sacrificial board probe (PID control, CS GP17) + oven probe (supervision, CS GP22) |
| HMI | SSD1306 OLED 0.96" 128×64 @ 0x3C, rotary encoder (GP10/GP11), Edit (GP12), Back (GP13) |
| Optional | HC-05 Bluetooth on UART1 (GP4/GP5, 115200 Bd) — telemetry mirror only, never in the control chain |
| Power | Certified 1 A USB charger → Schottky 1N5819 → VSYS (pin 39); the PC is data only |

Wiring details and BOM: `Electronics/Readme.md` and `Docs/InitialContext.md`.

### Pinout (frozen — as-built in `Electronics/PicoReflowOven.kicad_sch`, supersedes the draft in `Docs/InitialContext.md` §6)

| Function | GP (pin) | Notes |
|----------|----------|-------|
| TFT_SD | GP0 (1) | J7 (TFT display) |
| SPI_CS2 (TFT) | GP1 (2) | J7.4 — manual CS GPIO |
| TFT_RESET | GP3 (5) | J7.8 |
| UART1 TX / RX (HC-05) | GP4 / GP5 (6/7) | J8.2 / J8.3 @ 115200 Bd |
| HC-05 BL_ENABLE / BL_STATE | GP2 / GP7 (4/10) | J8.6 / J8.1 (AT-mode entry / link state) |
| Case fan (via NPN Q4) | GP6 (9) | R13 1k base + R14 10k pull-down, flyback D4, J5 |
| Buzzer (via NPN Q3) | GP8 (11) | R12 1k base + R11 10k pull-down, buzzer on VSYS (5 V), `#define` ACTIVE vs PWM (passive → ~100 Ω series) |
| Status LED | GP9 (12) | R10 330 → D3 → GND; solid = heating, blink = alarm, off = idle |
| Encoder A / B | GP10 / GP11 (14/15) | R8/R9 1k pull-ups + C3/C4 100 nF |
| **Edit** button | GP12 (16) | SW1 + R6 1k pull-up + C1 100 nF → GND |
| **Back** button | GP13 (17) | SW2 + R7 1k pull-up + C2 100 nF → GND |
| Reserve | GP14 (19) | free |
| Opto MOC (via NPN Q2) | GP15 (20) | R2 1k base + R3 10k pull-down → J2 (TriacConn) |
| SPI0 MISO / CS0 / SCK / MOSI | GP16 / GP17 / GP18 / GP19 (21/22/24/25) | 2× MAX6675 (J1, J6) + TFT (J7) |
| I2C0 SDA / SCL (OLED) | GP20 / GP21 (26/27) | J3 + R4/R5 4.7 kΩ pull-ups to 3V3, 0x3C |
| SPI CS1 (2nd MAX6675) | GP22 (29) | J6.4 — manual CS GPIO |
| Free | GP26–28 (31–34) | |
| Power | VSYS 39 / GND 38 / 3V3 36 | J4 (ext. USB-C module) → D2 1N5819 → VSYS; VBUS 40 → R1 4.7k → D1 LED → GND |

### Measurement & power

- 2× K-type probes + 2× MAX6675 on shared SPI0, read ≤ 4 Hz, open-TC detection built in
- **Board probe** (FR4 plate, thermocouple bolted across a drilled hole) = the PID control probe; **oven probe** = supervision with absolute cutoffs (~230 °C SnBi autonomous / ~300 °C SAC & bench)
- **Directional alarm:** while heating, air must lead the board (gap > ~60 °C = runaway); the inversion during cooling is normal
- **Uniformity calibration** (once): board probe in 3 positions; gradient < 5–8 °C at peak is acceptable
- **Power:** external USB-C module on J4 → Schottky D2 (1N5819) → VSYS (pin 39); VBUS (pin 40) → R1 → D1 power LED; PC = data only. Budget ~275 mA running / ~400 mA peak → ≥ 2.5× margin; optional 470–1000 µF reservoir cap

---

## Talking to it from a Linux PC

`/dev/ttyACM0` — native CDC, opening the port does **not** reset the Pico, so you can start capturing mid-cycle. Line-based text protocol.

| Command | Effect |
|---------|--------|
| `PROFILE <name>` | Pick a curve from the profile table (`SnBi standard`, `SnBi slow`, `SAC305 standard`, `manual`…) |
| `START` | Launch the cycle |
| `ABORT` | Immediate stop |
| `GAINS kp ki kd` | Set PID gains |
| `MODE MANUAL x%` | Manual mode, duty cycle x% |

Telemetry, 1 Hz CSV: `t,sp,tc_board,tc_air,duty,STATE` — mirrored on UART1 (HC-05 Bluetooth) if fitted. Both links are observers, never in the control chain.

- You'll need to be in group `dialout`; watch out for ModemManager (disable it, or add a udev rule `ID_MM_DEVICE_IGNORE`)
- Capture: `cat /dev/ttyACM0 | tee run.csv`, or a pyserial + matplotlib script for PID tuning
- **Nothing is logged before the port is opened.**

---

## The little screen on the box

- OLED SSD1306 0.96" 128×64 @ 0x3C, refreshed at 1 Hz
- Encoder = navigate / increment-decrement
- **Edit** = enter menu, edit, validate — and START when the cursor is on the Start field
- **Back** = exit / cancel
- Once started, the field becomes **STOP**: Edit OR Back = immediate stop (deliberate redundancy — stopping never depends on a single button). No long-press, no confirmation dialog.

---
>>>>>>> 6148751 (Updated schematics and wired components)

## Repo layout

```
PicoReflowOven/
├── Readme.md                  ← you are here
├── AGENTS.md                  ← rules for AI agents working in this repo
├── Docs/
│   ├── InitialContext.md      ← full project write-up (Nov 2026, FR) — decision source of truth
│   └── DesignDeepDive.md      ← technical deep dive (oven analysis, safety, pinout, protocol…)
├── Electronics/
│   ├── Readme.md              ← HT daughter board, triac, wiring, BOM
│   └── PicoReflowOven/        ← KiCad schematic + PCB (pinout reference)
└── Firmware/
    └── Readme.md              ← firmware layers, pinout, USB protocol
```

## Where we're at (Nov 2026)

- [x] Design frozen (safety architecture, triac choice, pinout per KiCad schematic)
- [ ] Wiring the power section + protections
- [ ] Firmware generation (layer 1 → 2 → 3)
- [ ] No-load runs (`MODE MANUAL 100` 20–30 min → unplugged thermal check → heatsink ≤ 70 °C)
- [ ] Calibration (probe in 3 positions), profile runs with PC plot, PID tuning
- [ ] First real reflow, logged

Still [OPEN]: SAC ramp near 250 °C (measure on first bench run) · SAC bench-only decision to be
logged as DEC-XXX · TME delivery checks (Kapton adhesive = silicone, TKD wire ≥ 1 mm², triac =
BTA316Y) · blower-on-cutout idea under exploration (see `Docs/DesignDeepDive.md` §1.1).

Later, maybe: re-insulate with rock wool (encapsulated) for a faster ramp and higher ceiling.

## Disclaimer

This thing plugs into the wall (230 V) and runs hot. Build at your own risk — the safety
architecture above is part of the design, don't bypass it.
