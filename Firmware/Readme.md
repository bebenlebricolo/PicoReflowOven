# Firmware

This folder will host the Pico firmware (C++ / pico-sdk) for the main board of this project.

The firmware is structured in **3 layers**. The pinout and HMI model are **frozen** — see below.
**Pinout reference: the KiCad schematic `Electronics/PicoReflowOven/PicoReflowOven.kicad_sch`**
(net labels at the Pico pins). Decision history: `Docs/InitialContext.md`; integrated specs:
`Docs/DesignDeepDive.md`.

---

## Layer Architecture

```
┌─────────────────────────────────────────────┐
│  Layer 3 — Local HMI (OLED + encoder + btn) │
├─────────────────────────────────────────────┤
│  Layer 2 — USB CDC protocol (observer only) │
├─────────────────────────────────────────────┤
│  Layer 1 — Control + safety (autonomous)    │
│  State machine, PID 1 Hz, slow PWM, cutoffs │
└─────────────────────────────────────────────┘
```

**Golden rule:** Layer 1 runs standalone. Layers 2 and 3 are observers/controllers — a USB disconnect or HMI failure never stops a running cycle.

---

## Layer 1 — Control + Safety (autonomous)

| Item | Spec |
|------|------|
| State machine | `RAMP → SOAK → PEAK → COOL`, plus `ALARM` |
| Control | PID at **1 Hz** on the sacrificial board probe |
| PWM | Slow PWM, period 200 ms – 5 s |
| Cutoffs | **Profile-dependent:** board probe ~230 °C (SnBi autonomous) / ~300 °C ceiling (bench & SAC); oven probe absolute cutoff. Never apply SnBi cutoffs to a SAC run |
| Directional alarm | In heating, air must lead board (gap > ~60 °C = runaway); inversion is normal in cooling |
| Buzzer | Abstracted via `#define` (ACTIVE DC vs PASSIVE PWM — component from salvage). Active 2-pin buzzer: direct NPN drive. Passive 8 Ω / piezo: PWM + ~100 Ω series resistor (NPN limit ~50 mA) |
| Debounce | Hardware (100 nF caps) → firmware uses simple edge detection, no debounce code |
| Quadrature | State-table decoding (IRQ or polling ≥ 1 kHz) |

---

## Layer 2 — USB Protocol (CDC)

- Device: `/dev/ttyACM0` on Linux (native cdc_acm, **no reset on port open** — capture can start mid-cycle)
- Line-based text protocol

### Commands
| Command | Effect |
|---------|--------|
| `PROFILE <name>` | Select a curve from the **extensible profile table** (`SnBi standard`, `SnBi slow`, `SAC305 standard`, `manual`…). New profiles are added per solder paste alloy / board type — store them in a dedicated table (e.g. `profiles.h`), do not hard-code a fixed list. Custom profile upload over USB is a possible extension [OPEN] |
| `START` | Launch the cycle |
| `ABORT` | Immediate stop |
| `GAINS kp ki kd` | Set PID gains |
| `MODE MANUAL x%` | Manual mode, duty cycle x% |

### Telemetry (1 Hz CSV)
```
t,sp,tc_board,tc_air,duty,STATE
```

- Duplicated on **UART1** (HC-05 Bluetooth, 115200 Bd) at no extra cost — telemetry mirror only, never in the control chain
- **Nothing is logged before the port is opened.**

### Linux PC-side
- Group `dialout`; watch out for ModemManager (disable it or add udev rule `ID_MM_DEVICE_IGNORE`)
- Capture: `cat /dev/ttyACM0 | tee run.csv` or pyserial + matplotlib for PID tuning

---

## Layer 3 — Local HMI

| Element | Detail |
|---------|--------|
| Display | SSD1306 OLED 0.96" 128×64, address 0x3C (confirm via I2C scan), refresh 1 Hz (~25 ms frame at 400 kHz — no conflict with control loop) |
| Encoder | Navigate / increment-decrement |
| **Edit** button | Enter menu, edit, validate — and **START** when cursor is on the Start field |
| **Back** button | Exit / cancel |

**Interaction model:** select a pre-configured curve → optional adjustments → Start. **Once started, the field becomes STOP: Edit OR Back = immediate stop** (deliberate redundancy — stopping never depends on a single button). No long-press, no confirmation: stopping is the simplest gesture on the HMI.

---

## Pinout (frozen — reference: KiCad schematic net labels)

| Function | GP | Notes |
|----------|-----|-------|
| Opto MOC (`TriacDriver`, via NPN) | GP15 | base via 2.2–4.7 kΩ + 10 k pull-down |
| SPI0 SCK / MISO / MOSI | GP18 / GP16 / GP19 | 2× MAX6675 shared bus |
| CS board probe / CS oven probe / CS2 (`SPI_CS0/1/2`) | GP17 / GP22 / GP5 | PID control / supervision / spare (SD) |
| I2C0 SDA / SCL | GP20 / GP21 | SSD1306 OLED @ 0x3C |
| UART (HC-05 via J8) TX / RX | [OPEN] GP0/GP1 | Not yet labeled at the Pico in the schematic; GP0/GP1 free (UART0). GP4/GP5 are taken by `TFT_SD`/`SPI_CS2` — earlier docs listing UART on GP4/GP5 are superseded |
| Encoder A / B | GP10 / GP11 | 100 nF on A and B, quadrature decode |
| **Edit** button | GP12 | NO → GND + 100 nF |
| **Back** button | GP13 | NO → GND + 100 nF |
| Encoder push (reserve) | GP14 | free in the 2-button model |
| Status LED | GP9 | via 330 Ω — solid = heating, blink = alarm, off = idle |
| Buzzer | GP8 | via NPN. `#define` ACTIVE vs PWM: active 2-pin buzzer = direct NPN drive; passive 8 Ω speaker / piezo = PWM on GP8 **with ~100 Ω series resistor** (stay under the NPN's ~50 mA) |
| Case fan (option) | GP6 | NPN + flyback + 10 k pull-down, or direct on rail |
| Display option `TFT_RESET` / `TFT_SD` | GP3 / GP4 | SPI TFT + SD, see schematic |
| Power | pins 39/38/36 | VSYS / GND / 3V3 (probes) |

### Measurement
- 2× K-type probes + 2× MAX6675 on shared SPI0 (SCK GP18, MISO GP16; CS GP17 board, CS GP22 oven)
- Read rate ≤ 4 Hz (conversion 0.17–0.22 s), open-TC detection built in
- **Board probe** (FR4 sacrificial PCB, **thermocouple bolted across a drilled hole**, reproducible position) = PID control. Mechanical fixing → no Kapton limit on the probe, SAC-compatible; Kapton only for positioning/isolation on the target board
- **Oven probe** (original nut) = supervision: absolute cutoff ~230 °C SnBi autonomous / ~300 °C SAC & bench ceiling
- **Uniformity calibration** (once): board probe in 3 positions; acceptable gradient < 5–8 °C at peak

### Power
- Certified USB charger 1 A → Schottky 1N5819/SS34 → VSYS (pin 39)
- PC = data only (backup power via internal D1)
- Budget ~275 mA running / ~400 mA peak → margin ≥ 2.5×
- Optional 470–1000 µF reservoir cap

---

## Status

Firmware not yet generated. Generation order once hardware assembly is underway: **Layer 1 → Layer 2 → Layer 3**.
