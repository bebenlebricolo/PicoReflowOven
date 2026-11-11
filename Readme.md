# Pico Reflow Oven

This project is my own version of a custom reflow oven, based on my former kitchen oven.
It's designed around pre-existing hardware (the oven itself) and some components I had in the drawers. So this will explain the frankenstein style of this project, as for once I'm not sourcing components specifically for a project, but rather building one from spare parts I have at hands.

## What is it for?

It's a pretty basic reflow oven, pre-equipped with a high temperature fan (cool!) and it'll be used to reflow custom PCBs at moderate to (hopefully) high temperatures.
Because it has a built in fan and 2 set of heating elements (resistors) at the top and bottom, it'll help reflowing PCBs while keeping the heat distribution as even as possible (forced convection).

---

## The gist

I recalibrated a **1500 W / 230 V** convection oven and put a **Raspberry Pi Pico** (C++ / pico-sdk) in charge of it.

- **Main target:** low-temp paste **Sn42Bi58** (melts at 138 °C, peak 170–180 °C)
- **Maybe later:** standard **SAC305 / SnAgCu** (peak ~245–250 °C) — bench mode only for now. The 250 °C stock ceiling is the thermal cutout talking, not a hardware limit (~300 °C expected in bench mode [HYPOTHESIS, not measured])

Measured ramp rate: 36–40 °C/min (~0.6–0.67 °C/s). Fast enough — the real fight is the **overshoot at the peak**, not the climb.

---

## The one rule that matters

> The control loop and all the safety stuff live on the Pico. The PC never supervises the hot chain.

A run in progress survives a USB disconnect. PC / Bluetooth links are just observers (telemetry, settings, launch). Lose the link mid-run = the run keeps going.

---

## How it's built — 3 layers

```
┌─────────────────────────────────────────────┐
│  Layer 3 — Local HMI                        │
│  SSD1306 OLED 128×64 + rotary encoder       │
│  + 2 buttons (Edit / Back)                  │
├─────────────────────────────────────────────┤
│  Layer 2 — USB protocol (CDC /dev/ttyACM0)  │
│  Commands: PROFILE, START, ABORT, GAINS,    │
│  MODE MANUAL x%                             │
│  Telemetry CSV 1 Hz:                        │
│  t,sp,tc_board,tc_air,duty,STATE            │
├─────────────────────────────────────────────┤
│  Layer 1 — Autonomous control + safety      │
│  State machine: RAMP/SOAK/PEAK/COOL/ALARM   │
│  PID 1 Hz on sacrificial board probe        │
│  Slow PWM (200 ms – 5 s period)             │
│  Interlocks, logical cutoffs                │
│  Runs standalone — no PC, no HMI needed     │
└─────────────────────────────────────────────┘
```

| Layer | Role | Works without PC? |
|-------|------|:-----------------:|
| 1 — Control + safety | State machine, PID, PWM, cutoffs | Yes (standalone) |
| 2 — USB protocol | Text commands + CSV telemetry | Observer only |
| 3 — Local HMI | OLED menu, encoder, buttons | Yes (standalone) |

---

## Profiles

**SnBi first — that's the daily driver:**

| Phase | Setpoint |
|-------|----------|
| Free ramp | ~0.6 °C/s up to ~145 °C |
| Soak | 145–150 °C, 60–90 s (flux activation) |
| Peak | 170–180 °C |
| TAL | 30–60 s above 138 °C |
| Cooling | Door open + convection fan |

**SAC305 / SnAgCu — later** (industry-standard numbers [DATASHEET]):

| Phase | Setpoint |
|-------|----------|
| Free ramp | ~1–2 °C/s target up to ~150 °C (the oven does ~0.6–0.67 °C/s — slower is fine, just longer; expect even less near 250 °C [OPEN]) |
| Soak | 150–180 °C, 60–120 s |
| Peak | 245–250 °C — above the 250 °C stock ceiling (cutout-limited, not a hardware limit); bench mode only, overshoot control is critical |
| TAL | 30–60 s above 217 °C |
| Cooling | Door open + convection fan |

The profile list will grow with paste alloy and board size (thermal mass) — it's a table in firmware, not a hard-coded enum. Each entry records: name, paste alloy, target board type, phase table.

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
| MCU | Raspberry Pi Pico (RP2040), C++ / pico-sdk |
| Power switching | BTA316Y-800CTQ triac (16 A / 800 V, 3Q, IGT 35 mA, factory-isolated tab) driven by MOC3042M opto-triac via NPN low-side |
| Daughter board | Reused from my "NanoFridgeThermostat" project (MOC3042M + snubber R 360 Ω / C 10 nF, TE 5.08 mm terminals, J5/J6 headers) |
| Temperature | 2× K-type probes + 2× MAX6675 (shared SPI0): sacrificial board probe (PID control, CS GP17) + oven probe (supervision, CS GP20) |
| HMI | SSD1306 OLED 0.96" 128×64 @ 0x3C, rotary encoder (GP10/GP11), Edit (GP12), Back (GP13) |
| Optional | HC-05 Bluetooth on UART1 (GP4/GP5, 115200 Bd) — telemetry mirror only, never in the control chain |
| Power | Certified 1 A USB charger → Schottky 1N5819 → VSYS (pin 39); the PC is data only |

Wiring details and BOM: `Electronics/Readme.md` and `Docs/InitialContext.md`.

### Pinout (frozen — firmware uses this as-is)

| Function | GP | Notes |
|----------|-----|-------|
| Opto MOC (via NPN) | GP15 | base via 2.2–4.7 kΩ + 10 kΩ pull-down |
| SPI0 SCK / MISO | GP18 / GP16 | 2× MAX6675 shared bus |
| CS board probe / CS oven probe | GP17 / GP20 | PID control / supervision |
| I2C0 SDA / SCL | GP0 / GP1 | SSD1306 OLED @ 0x3C |
| UART1 TX / RX | GP4 / GP5 | HC-05 @ 115200 Bd |
| Encoder A / B | GP10 / GP11 | 100 nF on A and B, quadrature decode |
| **Edit** button | GP12 | NO → GND + 100 nF |
| **Back** button | GP13 | NO → GND + 100 nF |
| Encoder push (reserve) | GP14 | free in the 2-button model |
| Status LED | GP9 | via 330 Ω — solid = heating, blink = alarm, off = idle |
| Buzzer | GP8 | via NPN, `#define` ACTIVE vs PWM |
| Case fan (option) | GP6 | NPN + flyback + 10 kΩ pull-down, or direct on rail |
| Power | pins 39/38/36 | VSYS / GND / 3V3 (probes) |

### Measurement & power

- 2× K-type probes + 2× MAX6675 on shared SPI0, read ≤ 4 Hz, open-TC detection built in
- **Board probe** (FR4 plate, thermocouple bolted across a drilled hole) = the PID control probe; **oven probe** = supervision with absolute cutoffs (~230 °C SnBi autonomous / ~300 °C SAC & bench)
- **Directional alarm:** while heating, air must lead the board (gap > ~60 °C = runaway); the inversion during cooling is normal
- **Uniformity calibration** (once): board probe in 3 positions; gradient < 5–8 °C at peak is acceptable
- **Power:** certified 1 A USB charger → Schottky 1N5819 → VSYS (pin 39); PC is data only. Budget ~275 mA running / ~400 mA peak → ≥ 2.5× margin; optional 470–1000 µF reservoir cap

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

## Repo layout

```
PicoReflowOven/
├── Readme.md              ← you are here
├── Agents.md              ← rules for AI agents working in this repo
├── Docs/
│   └── InitialContext.md  ← the full project write-up (Nov 2026) — source of truth
├── Electronics/
│   └── Readme.md          ← HT daughter board, triac, wiring, BOM
└── Firmware/
    └── Readme.md          ← firmware layers, pinout, USB protocol
```

### The 9 reflexes

1. Cable gland + 1.5 mm² flexible on the outside only, crimped ferrules
2. No PVC in the cavity gap
3. Phase junctions outside the shell, in a closed box
4. Convection fan outside the triac loop
5. Triac tab: moderate torque + thermal paste (BTA isolated = plate not live)
6. HV/LV ≥ 6 mm apart, air through the heatsink
7. Open the serial capture **before** starting the cycle
8. First thermal check unplugged, at residual heat
9. Timer armed at every run + cutout reconnected after bench

---

## Where we're at (Nov 2026)

- [x] Design frozen (pinout, safety architecture, triac choice)
- [ ] Wiring the power section + protections (the only non-compressible step)
- [ ] Firmware generation (layer 1 → 2 → 3)
- [ ] No-load runs: `MODE MANUAL 100` 20–30 min → thermal check with power off → heatsink test (base ≤ 70 °C)
- [ ] Calibration: probe in 3 positions, then profile runs with PC plot
- [ ] PID tuning (overshoot at soak → peak)
- [ ] First real reflow, logged

Still [OPEN]:

- SAC ramp rate near 250 °C — to measure on the first SAC bench run (expect < 0.6 °C/s)
- SAC bench-only decision to be formally logged as DEC-XXX in `Docs/`
- TME delivery checks: Kapton adhesive = silicone, TKD wire ≥ 1 mm², triac = BTA316Y

---

## Later, maybe

- **Re-insulate with rock wool** instead of the original ceramic fiber (nasty stuff to handle). Should give a faster ramp and a higher ceiling (less loss) — but encapsulate it so no fibers ever get into the convection airflow.
- **More profiles** per paste alloy and board type — see `Firmware/Readme.md`.

## Disclaimer

This thing plugs into the wall (230 V) and runs hot. Build at your own risk — the safety architecture above is part of the design, don't bypass it.
