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

- **From the box:** OLED + encoder + Edit/Back. Edit = validate/START, Back = cancel; once
  running, the field becomes STOP and either button stops immediately.
- **From a Linux PC:** `/dev/ttyACM0` (native CDC, no reset on open — capture can start mid-cycle).
  Line protocol: `PROFILE <name>`, `START`, `ABORT`, `GAINS kp ki kd`, `MODE MANUAL x%`;
  1 Hz CSV telemetry `t,sp,tc_board,tc_air,duty,STATE` (e.g. `cat /dev/ttyACM0 | tee run.csv`).
  Group `dialout`; disable ModemManager. Nothing is logged before the port is opened. Spec:
  `Firmware/Readme.md`.
---

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
