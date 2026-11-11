# PicoReflowOven — Agent Guidelines

This file defines behavior and constraint rules for AI agents operating within this repository.

---

## Project Context

**PicoReflowOven** — DIY reflow oven built around a recalibrated **1500 W / 230 V domestic convection oven** controlled by a **Raspberry Pi Pico** (C++ / pico-sdk).

- **Primary target:** low-temperature solder paste **Sn42Bi58** (melting point 138 °C, peak 170–180 °C).
- **Secondary target (not excluded):** standard **SnAgCu / SAC305** profiles (melting point 217 °C, peak ~245–250 °C). SAC runs have specific safety and measurement implications — see *Safety Architecture* below. All procedures MUST state which profile family they apply to.

### Golden Rule
**The control loop and all safety mechanisms live on the Pico — the PC never supervises the hot chain.** A running cycle survives a USB disconnect; any PC/Bluetooth link is a passive observer (telemetry, settings, launch). Link loss during a run = the run continues.

### Architecture (3 layers)
1. **Layer 1 — Autonomous control + safety**: state machine (RAMP/SOAK/PEAK/COOL/ALARM), 1 Hz PID on the sacrificial board probe, slow PWM (200 ms – 5 s period), interlocks, logical cutoffs. Runs standalone, no PC or HMI needed.
2. **Layer 2 — USB protocol** (CDC, `/dev/ttyACM0`, line-based text): commands `PROFILE`, `START`, `ABORT`, `GAINS`, `MODE MANUAL x%`; 1 Hz CSV telemetry: `t,sp,tc_board,tc_air,duty,STATE`. Native CDC — no reset on port open, capture can start mid-cycle.
3. **Layer 3 — Local HMI**: SSD1306 OLED + rotary encoder + 2 buttons (Edit/Back).

### Supported Reflow Profiles

**SnBi (primary):**
| Phase | Setpoint |
|-------|----------|
| Free ramp | ~0.6 °C/s up to ~145 °C |
| Soak | 145–150 °C, 60–90 s |
| Peak | 170–180 °C |
| TAL | 30–60 s above 138 °C |
| Cooling | door open + convection fan |

**SAC305 / SnAgCu (secondary — industry-standard profile values [DATASHEET]):**
| Phase | Setpoint |
|-------|----------|
| Free ramp | ~1–2 °C/s target up to ~150 °C (oven measured ramp is ~0.6–0.67 °C/s at low temp; expect lower near 250 °C — [OPEN] to measure) |
| Soak | 150–180 °C, 60–120 s |
| Peak | 245–250 °C (above the 250 °C stock ceiling — cutout-limited, not a hardware limit; bench mode only, overshoot control is critical) |
| TAL | 30–60 s above 217 °C |
| Cooling | door open + convection fan |

> 250 °C is the **stock ceiling** imposed by the original thermal cutout, not a hardware limit. In bench mode (cutout disconnected) the cavity is expected to reach ~300 °C [HYPOTHESIS — user estimate, not measured; do not probe the absolute limit]. The SAC peak (~245–250 °C) therefore has thermal headroom in bench mode; overshoot control remains the key challenge. SAC is always a bench-grade operation (bench mode only — decided, see below).

### Profile Set Is Extensible
More profiles will be added over time, depending on **solder paste alloy** and **board type** (thermal mass). Treat the profile list as extensible — never hard-code a fixed SnBi/SAC-only set. Each profile must record: name, paste alloy, target board type, and the full phase table.

### Future Modifications (roadmap — [OPEN], not part of the current build)
- **Re-insulation with rock wool:** replace the original ceramic-fiber insulation with rock wool (ceramic fiber is hazardous to handle). Expected effect: higher ramp rate and higher ceiling (lower thermal losses). The insulation must be encapsulated so no fibers ever enter the convection airflow. Not started — do not document as done.

### Safety Architecture (non-negotiable)
- **No fuse, no thermal fuse** (deliberate decision — a shorted triac draws the same current as the load, no fuse sees that fault). Hardware bounding is done by:
  1. **Autonomous mode (SnBi only, default)**: original thermal cutout reinserted in series (~210–220 °C resettable, above the 180 °C SnBi peak, independent of Pico and triac). **This cutout trips below the SAC peak — it cannot protect a SAC run.**
  2. **Bench mode (any SAC run or any test > 250 °C, cutout disconnected)**: oven loaded per profile, sacrificial probe fixation rated for the target temperature, operator present at all times, mechanical timer armed, logical cutoffs ~300 °C.
  3. **Mechanical timer (auto-off) armed at every run** — master switch AND time bound.
  4. **After every bench/SAC session: reconnect the cutout before any SnBi autonomous cycle** (physical verification).
  5. **SAC safety path — DECIDED (Nov 2026, to be logged as DEC-XXX): SAC runs are bench-mode only.** The original ~210–220 °C cutout stays in series for SnBi autonomous mode and is disconnected for SAC. Do NOT attempt to shift the cutout's trip point by blowing fresh air on it — artificially cooling the thermal cutout defeats its protection; this idea was considered and **rejected, do not implement**.
- General rules: thermal checks only with power off (unplug at wall, touch at residual heat); nothing 3D-printed inside the cavity; **no PVC in the cavity gap**; HV/LV separation ≥ 6 mm in the enclosure, airflow through heatsink (intake low / exhaust high).
- **Sacrificial board probe fixation — RESOLVED:** the probe is **mechanically bolted through a drilled FR4 plate** (thermocouple bolted across the board) — no Kapton temperature limit on the probe itself, SAC-compatible. Kapton is used only for positioning/isolation on the target board during a run.

### Key Design Decisions (do not reopen)
- **Triac**: BTA316Y-800CTQ (WeEn, 16 A / 800 V, 3Q Hi-Commutation, IGT 35 mA, factory-isolated tab IITO-220). Drop-in replacement for FT0814MH (phase 1 only, bare tab, requires mica isolation kit).
- **Opto drive**: NPN low-side pattern — "the GPIO is a signal, not a source". LED chain: 5V_EXT → 330 Ω → J5 → MOC3042 LED → J6 → NPN collector → emitter → GND. Base driven by GP15 via 2.2–4.7 kΩ + 10 kΩ pull-down. Zero GPIO current budget for the LED (~11.4 mA on 5V_EXT rail).
- **SSR-25DA rejected** (mass counterfeits, same ~8 W dissipation at 6.5 A → heatsink needed anyway; genuine SSRs at 25–45 € buried).
- **No fan in the cavity gap** (PTFE 260 °C removes the need; blowing in would lower ceiling and ramp rate).

### Source of Truth
- `Docs/InitialContext.md` — complete project synthesis (November 2026), all design decisions frozen. Sections marked "écarté" (rejected) are not to be reopened unless new material facts emerge.

---

## Documentation Standards

### General Rules
- **Language:** Technical documentation in English; conversations in French unless requested otherwise
- **Style:** Clear, concise, factual, no marketing fluff
- **Structure:** Use markdown tables for comparisons; hierarchies for options

### Critical Rules
- NEVER claim something is "verified" or "tested" unless executed in this session
- ALWAYS distinguish: datasheet values vs. interpretations vs. hypotheses vs. open questions
- ALWAYS use `~` prefix for approximate values with source noted

### Fact Labeling
Label all technical claims with their source:
- **[DATASHEET]** — From manufacturer documentation
- **[INTERPRETED]** — Technical analysis based on datasheet values
- **[HYPOTHESIS]** — Educated guess requiring validation
- **[OPEN]** — Unknown, needs investigation

---

## File Operations

- **Read before edit:** Never edit a file you haven't read in the current session
- **Minimal changes:** Only modify what's explicitly requested; preserve existing style
- **Tool preference:** Use integrated tools (`read_file`, `write_file`, `edit`, `grep`) over shell commands when possible
- **Path format:** Use absolute paths (`/c/Users/...` in Git Bash)

---

## Git Rules

- **Commit messages:** Imperative mood, 50-72 char subject line
- **Push:** Never push without explicit user approval
- **Branches:** Use descriptive names
- **CRITICAL:** NEVER create git commits autonomously without explicit user request — ALWAYS ask for permission first

---

## Documentation Standards

### Source of Truth
- **Decision Logs and Specs are authoritative:** All Decision Logs (DEC-XXX format) and technical specifications written by users are considered primary sources of truth. LLMs MUST use these documents to reconstruct context and understand project decisions.
- **`Docs/InitialContext.md`** is the single source of truth for the project state. Use it to write assembly, usage, or maintenance documentation without re-reading design history.

---

## Project-Specific Rules

- **Safety-critical content:** Never downplay or omit the safety architecture (thermal cutout, mechanical timer, bench mode conditions). Any document touching wiring, power, or operation MUST reference the applicable safety rules.
- **Rejected decisions are closed:** Do not propose SSR-25DA, fuses/thermal fuses, cavity-gap fans, or reopening the triac choice unless new material facts emerge. Mark them as "rejected — do not reopen" if mentioned.
- **Pinout is frozen:** The GPIO assignment in `Docs/InitialContext.md` §6 is final. Firmware generation must use it as-is.
- **Autonomy of the Pico:** Never design a feature where the PC supervises the hot chain. PC/Bluetooth links are observers only.
- **Profile-dependent cutoffs:** Firmware cutoffs MUST follow the selected profile and mode — SnBi autonomous ~230 °C board-probe cutoff (with the ~210–220 °C hardware cutout in series); SAC/bench ceiling ~300 °C logical. Never apply the SnBi cutoff to a SAC run, nor run SAC with the SnBi cutout connected.
- **SAC = bench mode only (decided):** every SAC run requires bench conditions — cutout disconnected, operator present, mechanical timer armed, logical cutoffs ~300 °C. Never defeat or bypass the thermal cutout (e.g. forced air cooling to shift its trip point — rejected).
- **Two profile families:** SnBi (primary) and SAC305 (secondary). Every procedure, log, and run record must state which profile it applies to.
- **Language of code comments and identifiers:** English. French only in user-facing conversation.