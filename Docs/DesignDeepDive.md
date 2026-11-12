# Design Deep Dive — PicoReflowOven

Technical companion to `Readme.md`. The Readme is the overview; this document carries the
technical detail. Decision history and rationale: `Docs/InitialContext.md` (French, source of
truth for decisions). **Pinout reference: `Electronics/PicoReflowOven/PicoReflowOven.kicad_sch`.**
BOM and wiring detail: `Electronics/Readme.md`. Firmware layer specs: `Firmware/Readme.md`.

Fact labels: **[DATASHEET]** manufacturer data · **[INTERPRETED]** analysis of datasheet values ·
**[HYPOTHESIS]** guess to validate · **[OPEN]** unknown / in progress. `~` = approximate.

---

## 1. Stock oven behavior

- 1500 W / 230 V domestic convection oven, resistive current ~6.5 A [measured], forced-convection
  fan, 2 sets of heating elements (top + bottom).
- Stock setpoint dial goes to ~230 °C, but the oven only ever reached ~190–200 °C [measured]:
  the original thermocontact sits in a space that heats up faster than the cooking cavity, so it
  trips early.
- With the steel case removed, the stock oven reached ~250 °C [measured].
- Ramp rate with a prior ceramic isolation cover: ~36–40 °C/min (~0.6–0.67 °C/s) [measured].
  Typical paste datasheets recommend ~1–3 °C/s [DATASHEET] — slower is workable, just longer.
  The real challenge is the **overshoot at the peak**, not the climb.

### 1.1 Blowing air on the thermocontact — [OPEN] exploration path (not rejected, not implemented)

Blowing fresh air on the thermocontact offsets its trip point higher. Note that a better cavity
insulation produces the exact same effect (same temperature-gradient change): insulation alone
took the oven from ~190 °C to ~250 °C peak [measured]. The idea is kept as a possible elegant
way to reach higher temperatures without replacing the thermocontact — **currently an open
exploration path, not implemented**.

Failure modes to design against if explored:

- With the blower running, a shorted-triac runaway may never trip the cutout — the cutout is the
  only hardware bound for that fault (no fuse sees it, see §3). The effective trip threshold then
  depends on the blower staying on.
- If the blower fails, the cutout trips earlier than with the blower on — that direction is
  fail-safe.

Mitigations considered (software): runaway detection in firmware (heater off + temperature still
rising = runaway); optional current transformer on the triac line to read the actual current
(pattern already implemented in the NanoFridgeThermostat project, reusable later); or simply
cutting the blower's supply so the cutout can do its job.

---

## 2. Mains path and the two hardware configurations

Stock chain: wall socket → mechanical timer (with bell) → original thermal cutout → heating-element
selector (top / bottom / both) → elements. Mode selector (natural / forced convection) is on a
parallel circuit right behind the timer; the convection fan runs on direct phase, **outside the
triac loop**.

Only one rerouting point is needed, before the element selector:

| Configuration | Wiring | Meaning |
|---|---|---|
| **Autonomous** (default, SnBi) | Triac inserted **before** the cutout; cutout stays in series | Cutout remains an independent hardware bound (~210–220 °C resettable) |
| **Bench** (any SAC run or test > 250 °C) | Cutout **disconnected**, its 2 wires routed to the triac instead | No mechanical thermal bound left → operator must be present, see §3 |

A 2-pin inline connector on the element supply allows returning to the stock configuration in
~30 s. Full chain: wall → timer → cutout (autonomous mode) → cable gland → J2 → triac → J3 →
cable gland → elements. Details: `Electronics/Readme.md`.

---

## 3. Safety architecture (non-negotiable)

Mains AC (230 V) is switched. Never touch live components. All active parts live outside the oven
enclosure; only 1.5 mm² wires (cutout leads) and thermocouple cables cross the shell, through
cable glands, into an external 3D-printed enclosure with a fan over the triac heatsink.

**No fuse, no thermal fuse** [decided]: a triac shorted closed draws the same current as the load
(~6.5 A) — no fuse would see that fault. Hardware bounding is done by:

1. **Autonomous mode (SnBi only, default):** original thermal cutout back in series
   (~210–220 °C resettable, above the 180 °C SnBi peak, independent of the Pico and the triac).
   It trips below the SAC peak → it cannot protect a SAC run. SnBi only.
2. **Bench mode (any SAC run or any test > 250 °C, cutout disconnected):** oven loaded per profile,
   sacrificial probe fixation rated for the target temperature, **operator present at all times**,
   mechanical timer armed, logical cutoffs ~300 °C.
3. **Mechanical timer armed at every run** — master switch AND time bound. Any runaway dies with
   the dial.
4. **After every bench/SAC session: physically reconnect the cutout** before any SnBi autonomous
   cycle.
5. **SAC = bench mode only** (decided Nov 2026, to be logged as DEC-XXX in `Docs/`).

House rules:

- Thermal checks only with the plug out of the wall, at residual heat.
- **No PVC/ABS or any thermoplastic in the cavity gap** (> 100 °C near the wall); nothing 3D-printed
  inside the cavity.
- HV/LV separated by ≥ 6 mm in the enclosure; airflow through the heatsink (intake low, exhaust
  high).
- Sacrificial probe **bolted through a drilled FR4 plate** — mechanical fixing, no Kapton limit on
  the probe itself, SAC-compatible. Kapton (silicone adhesive) only for positioning/isolation on
  the target board.

Assembly reflexes (9): cable gland + 1.5 mm² flexible outside only, crimped ferrules · no PVC in
the cavity gap · phase junctions outside the shell in a closed box · convection fan outside the
triac loop · triac tab moderate torque + thermal paste (BTA isolated tab = plate not live) ·
HV/LV ≥ 6 mm, air through the heatsink · open the serial capture **before** starting the cycle ·
first thermal check unplugged, at residual heat · timer armed at every run + cutout reconnected
after bench. Full wiring procedure: `Electronics/Readme.md`.

---

## 4. Architecture — 3 layers, one golden rule

**Golden rule:** the control loop and all safety mechanisms live on the Pico — the PC never
supervises the hot chain. A run in progress survives a USB disconnect; any PC/Bluetooth link is a
passive observer (telemetry, settings, launch). Link loss during a run = the run continues.

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

## 5. Hardware — mostly from the drawers

| Block | Detail |
|-------|--------|
| MCU | Raspberry Pi Pico (RP2040), C++ / pico-sdk — chosen over a Nano because the Nano's DTR reset on serial-open would kill a running cycle; Pico also adds a second UART (HC-05) and 264 KB RAM (full OLED framebuffer) |
| Power switching | BTA316Y-800CTQ triac (16 A / 800 V, 3Q Hi-Commutation, IGT 35 mA, factory-isolated IITO-220 tab) [DATASHEET], driven by MOC3042M opto-triac via NPN low-side — "the GPIO is a signal, not a source": LED chain `5V_EXT → 330 Ω → J5 → MOC3042 LED → J6 → NPN collector → GND`, base on GP15 via 2.2–4.7 kΩ + 10 kΩ pull-down, ~11.4 mA on the 5V_EXT rail, zero GPIO budget |
| Daughter board | Reused from the NanoFridgeThermostat project (MOC3042M + snubber R 360 Ω / C 10 nF, TE 5.08 mm terminals, J5/J6 headers); revisited schematic in `Electronics/PicoReflowOven/` |
| Temperature | 2× K-type probes + 2× MAX6675 on shared SPI0: sacrificial board probe (PID control) + oven probe (supervision) |
| HMI | SSD1306 OLED 0.96" 128×64 @ 0x3C, rotary encoder, Edit / Back buttons |
| Optional | HC-05 Bluetooth on a UART — telemetry mirror only, never in the control chain |
| Power | Certified 1 A USB charger → Schottky 1N5819 → VSYS (pin 39); the PC is data only |

Rejected — do not reopen: SSR-25DA (mass counterfeits, same ~8 W dissipation at 6.5 A → heatsink
needed anyway), fuses / thermal fuses (see §3), fan in the cavity gap (PTFE 260 °C removes the
need; blowing in would lower ceiling and ramp rate), FT0814MH for the final build (phase-1
validation only; bare tab needs a mica kit).

---

## 6. Pinout (reference: KiCad schematic `Electronics/PicoReflowOven/PicoReflowOven.kicad_sch`)

Extracted from the schematic net labels at the Pico pins:

| Net (schematic) | GP | Function | Notes |
|-----------------|-----|----------|-------|
| `TriacDriver` | GP15 | Opto MOC via NPN | base via 2.2–4.7 kΩ + 10 kΩ pull-down |
| `SPI_CLK` / `SPI_MISO` / `SPI_MOSI` | GP18 / GP16 / GP19 | SPI0 to the MAX6675s | shared bus |
| `SPI_CS0` / `SPI_CS1` / `SPI_CS2` | GP17 / GP22 / GP5 | CS board probe / CS oven probe / spare (SD) | PID control / supervision |
| `I2C_SDA` / `I2C_SCL` | GP20 / GP21 | I2C0, OLED SSD1306 @ 0x3C | |
| `UART_TX` / `UART_RX` (HC-05 via J8) | [OPEN] | Bluetooth telemetry mirror | not yet labeled at the Pico in the schematic; GP0/GP1 free (UART0). Note: GP4/GP5 are taken (`TFT_SD` / `SPI_CS2`) |
| `EncoderA` / `EncoderB` | GP10 / GP11 | Rotary encoder | 100 nF on A and B |
| `EditButton` | GP12 | Edit / validate / START·STOP | NO → GND + 100 nF |
| `BackButton` | GP13 | Back / cancel / STOP | NO → GND + 100 nF |
| — | GP14 | Encoder push (reserve) | free in the 2-button model |
| (wire) | GP9 | Status LED | via 330 Ω — solid = heating, blink = alarm, off = idle |
| `Buzzer` | GP8 | Buzzer via NPN | `#define` ACTIVE vs PWM (passive → PWM + ~100 Ω series, NPN limit ~50 mA) |
| `CaseFan` | GP6 | Case fan (option) | NPN + flyback + 10 kΩ pull-down, or direct on rail |
| `TFT_RESET` / `TFT_SD` | GP3 / GP4 | Display option (SPI TFT + SD) | see schematic |
| — | pins 39/38/36 | VSYS / GND / 3V3 (probes) | |

> Earlier documents (`Docs/InitialContext.md` §6) listed I2C on GP0/GP1, CS-oven on GP20 and UART on
> GP4/GP5. The KiCad schematic is the reference and supersedes those: I2C is on GP20/GP21, CS-oven
> on GP22. Firmware generation must use the schematic as-is.

---

## 7. Measurement & power

- 2× K-type probes + 2× MAX6675 on shared SPI0, read ≤ 4 Hz (conversion 0.17–0.22 s), open-TC
  detection built in.
- **Board probe** (FR4 plate, thermocouple bolted across a drilled hole) = the PID control probe.
  **Oven probe** = supervision with absolute cutoffs: ~230 °C SnBi autonomous / ~300 °C SAC &
  bench. **Profile-dependent cutoffs: never apply the SnBi cutoff to a SAC run, and never run SAC
  with the SnBi cutout connected.**
- **Directional alarm:** while heating, air must lead the board (gap > ~60 °C = runaway); the
  inversion during cooling is normal.
- **Uniformity calibration** (once): board probe in 3 positions; gradient < 5–8 °C at peak is
  acceptable.
- **Power:** certified 1 A USB charger → Schottky 1N5819/SS34 → VSYS (pin 39); PC is data only.
  Budget ~275 mA running / ~400 mA peak → ≥ 2.5× margin; optional 470–1000 µF reservoir cap.

---

## 8. USB / Bluetooth protocol (Layer 2 — observer only)

`/dev/ttyACM0` — native CDC; opening the port does **not** reset the Pico, so capture can start
mid-cycle. Line-based text protocol.

| Command | Effect |
|---------|--------|
| `PROFILE <name>` | Pick a curve from the extensible profile table (`SnBi standard`, `SnBi slow`, `SAC305 standard`, `manual`…) |
| `START` | Launch the cycle |
| `ABORT` | Immediate stop |
| `GAINS kp ki kd` | Set PID gains |
| `MODE MANUAL x%` | Manual mode, duty cycle x% |

Telemetry, 1 Hz CSV: `t,sp,tc_board,tc_air,duty,STATE` — mirrored on the HC-05 UART if fitted. Both
links are observers, never in the control chain. **Nothing is logged before the port is opened.**

PC side (Linux): be in group `dialout`; watch out for ModemManager (disable it or add a udev rule
`ID_MM_DEVICE_IGNORE`); capture with `cat /dev/ttyACM0 | tee run.csv`, or a pyserial + matplotlib
script for PID tuning.

---

## 9. Local HMI model (Layer 3)

- OLED SSD1306 0.96" 128×64 @ 0x3C, refreshed at 1 Hz.
- Encoder = navigate / increment-decrement.
- **Edit** = enter menu, edit, validate — and START when the cursor is on the Start field.
- **Back** = exit / cancel.
- Once started, the field becomes **STOP**: Edit OR Back = immediate stop (deliberate redundancy —
  stopping never depends on a single button). No long-press, no confirmation dialog.

---

## 10. Reflow profiles

**SnBi (primary — daily driver):**

| Phase | Setpoint |
|-------|----------|
| Free ramp | ~0.6 °C/s up to ~145 °C |
| Soak | 145–150 °C, 60–90 s (flux activation) |
| Peak | 170–180 °C |
| TAL | 30–60 s above 138 °C |
| Cooling | Door open + convection fan |

**SAC305 / SnAgCu (secondary — bench mode only, industry-standard values [DATASHEET]):**

| Phase | Setpoint |
|-------|----------|
| Free ramp | ~1–2 °C/s target up to ~150 °C (oven measured ~0.6–0.67 °C/s; expect less near 250 °C [OPEN]) |
| Soak | 150–180 °C, 60–120 s |
| Peak | 245–250 °C — above the ~250 °C stock ceiling (cutout-limited, not a hardware limit); bench mode only, overshoot control is critical |
| TAL | 30–60 s above 217 °C |
| Cooling | Door open + convection fan |

The profile list is **extensible** (paste alloy × board type / thermal mass): a table in firmware,
not a hard-coded enum. Each entry records: name, paste alloy, target board type, full phase table.

---

## 11. Status (Nov 2026) and open items

- [x] Design frozen (safety architecture, triac choice; pinout per KiCad schematic)
- [ ] Wiring the power section + protections (the only non-compressible step)
- [ ] Firmware generation (layer 1 → 2 → 3)
- [ ] No-load runs: `MODE MANUAL 100` 20–30 min → thermal check with power off → heatsink test
  (base ≤ 70 °C)
- [ ] Calibration: probe in 3 positions, then profile runs with PC plot
- [ ] PID tuning (overshoot at soak → peak)
- [ ] First real reflow, logged

Still [OPEN]:

- SAC ramp rate near 250 °C — measure on the first SAC bench run (expect < 0.6 °C/s)
- SAC bench-only decision to be formally logged as DEC-XXX in `Docs/`
- TME delivery checks: Kapton adhesive = silicone, TKD wire ≥ 1 mm², triac = BTA316Y (the Y =
  isolated tab)
- UART pins for the HC-05 not yet labeled at the Pico in the schematic (GP0/GP1 free)
- Blower-on-thermocontact idea (§1.1) — exploration path with the software mitigations listed there

Later, maybe: re-insulate with rock wool instead of the original ceramic fiber (encapsulated so no
fibers enter the convection airflow; expected faster ramp + higher ceiling) — not started.
