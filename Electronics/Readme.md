# Electronics

This folder is the home of the Electronics designs (schematics, PCBs, BOM, wiring).

The power section reuses a daughter board from the **NanoFridgeThermostat** project and drives a **230 V / 1500 W** convection oven through a triac. All design decisions below are frozen — see `Docs/InitialContext.md` for the full rationale.

---

## HT Daughter Board (reused from NanoFridgeThermostat)

Reusing the HT Daughter board from [NanoFridgeThermostat:](https://github.com/bebenlebricolo/NanoFridgeThermostat)
| Item | Value |
|------|-------|
| Opto-triac | **MOC3042M** (zero-cross detection built in) |
| Snubber | R4 360 Ω + C1 10 nF / 500 V AC |
| Terminals | TE 5.08 mm |
| Headers | J5 = `TriacDriverEnable`, J6 = LED return (2.54 mm) |
| R8 | 330 Ω (unchanged — design point: limits MOC peak current to 1 A) |

**J6 is no longer ground** — it is the switched LED return. At rest, J6 measures ~5 V; that is normal, not a fault.

---

## Triac

| | Phase 1 (validation only) | Final |
|---|---|---|
| Part | FT0814MH | **BTA316Y-800CTQ** (WeEn) |
| Rating | 8 A / 600 V, bare tab | 16 A / 800 V, **IITO-220 factory-isolated tab** (2500 V RMS) |
| IGT | 35 mA, 3Q | 35 mA, 3Q Hi-Commutation — drop-in proven |
| Isolation | Mica kit mandatory (SMICA-TO220-2 + NIPPEL-TO220), nylon screw or insulating shoulder — the screw passes through tab AND plate | None needed — tab is isolated; plate and enclosure never live |

TO-220 pinout: 1 = MT1, 2 = MT2, 3 = G.

Dissipation at 6.5 A: ~7.5–8.5 W → heatsink required, target Rth ≤ 3 °C/W. Tj ≈ 75 °C at full power on plate/heatsink in airflow.

**Heatsink candidates** (empirical test: `MODE MANUAL 100` for 20–30 min, base ≤ 70 °C wins):
- Alu plate 100×100 mm (salvaged)
- Fischer SK610-84AL
- Alutronic PR33/25.4/SE

Mount vertically, air on both faces, thermal paste, moderate screw torque.

---

## Opto Drive — NPN Low-Side (retained pattern)

> **The GPIO is a signal, not a source.**

LED chain: `5V_EXT → 330 Ω → J5 → MOC3042 LED → J6 → NPN collector → emitter → GND`

- Base driven by **GP15** via 2.2–4.7 kΩ, **10 kΩ pull-down** (Hi-Z at boot = explicit OFF)
- Base current ~1 mA — default 4 mA GPIO is enough, no drive strength needed
- LED ~11.4 mA on the 5V_EXT rail → **zero I/O budget** on the Pico (total budget ≈ 10 mA, no brown-out risk)
- Non-inverting: GP15 high = heat
- Any salvaged NPN: BC547 / 2N3904 / 2N2222
- Same NPN pattern for buzzer (GP8, base via 1 kΩ) and optional case fan (GP6, base via 470 Ω + 10 k pull-down)
- Check J5 polarity with a multimeter (diode mode, ~1.1 V forward)

---

## Mains Chain

```
Wall socket → mechanical timer → original thermal cutout (autonomous mode)
  → cable gland → J2 → triac → J3 → cable gland → heating elements
```

- Flexible 1.5 mm², crimped ferrules, cable ties
- **Convection fan powered continuously, outside the triac loop** (direct phase from the timer)
- PCB traces (IPC-2221, 1 oz): phase 4 mm, neutral 5.8 mm

---

## Wiring & Cables

| Zone | Cable | Notes |
|------|-------|-------|
| Cavity gap (inside shell) | **LAPP ÖLFLEX HEAT 350 SC 1.5 mm²** (glass fiber, 350 °C) or factory wire rated 250 °C+ | **Crimp (nickel core), never solder** — tin melts at 230 °C. Silicone 180 °C forbidden here. Alt: PTFE 260 °C (ÖLFLEX HEAT 260 SC) |
| Outside | Silicone 180 °C, 1.5 mm² (TKD) — verify cross-section ≥ 1 mm² at delivery |
| Element terminals | Bare spade terminal + Kapton with **silicone adhesive** (~280 °C) or PTFE heat-shrink 260 °C | Standard acrylic adhesive peels at ~155 °C |

- PVC-to-factory-wire junction **outside** the shell, crimped splice in a closed box; cable glands on the outside (BM4007 PG7, IP68)
- Every splice under glass-fiber sleeving
- Drill the shell as close as possible to the element terminals; 2-pin inline connector on the element supply (return to stock config in 30 s)

### 9 assembly reflexes
1. Cable gland + 1.5 mm² flexible outside only, crimped ferrules
2. No PVC in the cavity gap
3. Phase junctions outside the shell, in a closed box
4. Convection fan outside the triac loop
5. Triac tab: moderate torque + paste (BTA isolated = plate not live)
6. HV/LV separation ≥ 6 mm, airflow through heatsink
7. Open the serial capture **before** starting the cycle
8. First thermal check with power off, at residual heat
9. Timer armed at every run + cutout reconnected after bench

### No-load test before reassembly
1. Timer OFF → ohmmeter on the loop (continuity timer → cutout → elements ≈ 35 Ω; triac blocked = open circuit)
2. `MODE MANUAL 100` for 20–30 min
3. Unplug at the wall → residual-heat check (triac/plate, terminals, cable glands, cable against sheet metal)

---

## BOM

### In stock
Pico · NanoFridgeThermostat daughter board · FT0814MH (phase 1) · 2× K probes + MAX6675 · encoders, buttons, LED · SSD1306 OLED · 10 A relay (backup) · HC-05 + HC-06 · 100×100 alu plate · USB charger 1 A · salvaged NPN · prototyping boards

### TME order (Nov 2026, ~41 € TTC)

| Item | Qty | Role |
|------|-----|------|
| Triac **BTA316Y-800CTQ** (the Y = isolated tab) | ×5 | switching + lifetime spares |
| LAPP ÖLFLEX HEAT 350 SC 1.5 mm² | 2 m | cavity gap, crimp |
| Polyimide tape IPT POL-12MM 12 mm × 33 m | 1 | Kapton — **verify silicone vs acrylic adhesive** |
| Ninigi SMICA-TO220-2 (mica) + NIPPEL-TO220 | ×10 each | phase-1 isolation kit (FT0814MH bare tab) |
| Heatsinks Fischer SK610-84AL + Alutronic PR33/25.4/SE | 1 + 1 | candidates vs salvaged alu plate |
| Cable glands BM4007 (PG7, IP68) | ×5 | shell outlet |
| 1N5819 Schottky | ×5 | power supply |
| TKD silicone cable 2 colors | 2×1 m | outside — **verify ≥ 1 mm²** |
