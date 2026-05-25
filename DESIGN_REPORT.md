# SG-SGMB Design Status Report

**Project:** SpiderGwen MotherBoard (SG-SGMB)
**MCU:** ESP32-S3-WROOM-1
**Date:** 2026-05-25
**Branch:** main @ f8494a6

---

## Overview

The SG-SGMB is the embedded controller board for the SpiderGwen hexapod. It handles real-time servo control (18× ST3215 half-duplex UART), battery management (2S Li-Ion), and sensor interfacing — offloading these tasks from the Raspberry Pi brain over SPI2.

---

## Schematic Structure

The design uses a hierarchical 5-sheet layout:

| Sheet | File | Status |
|---|---|---|
| ESP32-S3 | `ESP32-S3.kicad_sch` | Done |
| Peripherals | `peripherals.kicad_sch` | Done |
| RPi Interface | `RPi_interface.kicad_sch` | Done |
| Programmer | `programmer.kicad_sch` | New (untracked) |
| Power Routes | `power_routes.kicad_sch` | In progress |

---

## Sheet-by-Sheet Status

### ESP32-S3
- ESP32-S3-WROOM-1U placed (U1) with full decoupling (C1 1uF, C2 10uF, C3 0.1uF)
- IO0 strapping pull-up (R1 10kΩ) for boot mode control
- EN enable RC circuit (R2 0Ω, configurable)
- JTAG debug header (J9, 4-pin 2.54mm)
- Two free IO breakout headers (J10, J11 — 4-pin each)
- Dedicated `+ESP_3V3` rail isolated from main `+3V3`

### Peripherals
- 6× foot microswitches — HCTL HC-5264-3A connectors (J1–J6) with 390nF debounce caps and 100nF bypass
- SW2, SW3, SW4, SW6 push switch footprints placed
- SN74LVC2G241 half-duplex buffer (SN74LVC2G1) — controls TX/RX direction for ST3215 servo UART bus
- MPU-6050 IMU with full decoupling network (C6 100nF, C59 10nF, C60 0.1uF, C10 390nF)
- Servo power rails `+7V4A`, `+7V4B`, `+7V4C` distributed to servo connectors

### RPi Interface
- RPi4 Model B 2×20 stacking header (J1 — PinHeader_2x20_P2.54mm)
- Bulk decoupling: C5 470uF + C4 10uF on 5V rail
- `+5V` and `GND` rail connections to RPi header

### Programmer *(new, untracked)*
- CP2102N USB-UART bridge (U8 — QFN-28) for flashing over USB-C
- USB-C receptacle (J8 — GCT USB4085) with ESD protection: D2, D3, D4 (LESD5D5.0CT1G, SOD-523)
- CC resistors R33, R34 (5.1kΩ) for USB-C sink detection
- Auto-reset circuit: Q2, Q3 (LRC L8050QLT1G NPN) with RTS/DTR control for ESP32 EN + IO0
- BOOT (SW7) and RESET (SW8) tactile switches (SW_SPST, PTS647)
- VBUS → +5V path with AO3401A load switch (Q4, SOT-23)
- Bulk caps on VBUS: C52, C54 (0.1uF/50V) + C53 (1uF/16V)

### Power Routes *(in progress)*
- Battery input: J7 XT30PW-M connector (`+VBAT`, 7.4V 2S Li-Ion)
- AO3401A P-FET (Q1) — reverse polarity protection on VBAT
- **3× TPS5683x buck converters** (U5, U6, U7 — VQFN-HR-10):
  - U5 → `+7V4A` (servo rail A, legs 1–2)
  - U6 → `+7V4B` (servo rail B, legs 3–4)
  - U7 → `+7V4C` (servo rail C, legs 5–6)
  - Each with 4.7uH inductors (L4, L5) and 22uF×2 output caps
- **SGM2212_3.3 LDO** (U2 — SOT-223) → `+3V3` from VBAT (logic/RPi supply)
- Power LED (D1 — 0201 LED) on power-good indicator
- `+5V` rail connected (sourced from RPi VBUS or USB-C VBUS TBD)

---

## Components Summary

| Category | Count (approx.) |
|---|---|
| Resistors | ~90 |
| Capacitors | ~60 |
| ICs / Modules | ~10 |
| Inductors | 5 |
| MOSFETs / Transistors | ~5 |
| Diodes / TVS | ~4 |
| Connectors | ~15 |
| Switches | ~10 |
| **Total** | **~130** |

**Key ICs:**

| Ref | Part | Function |
|---|---|---|
| U1 | ESP32-S3-WROOM-1U | Main MCU |
| U2 | SGM2212_3.3 | 3.3V LDO |
| U5/U6/U7 | TPS5683x | 3× Buck converters (7.4V servo rails) |
| U8 | CP2102N-Axx-xQFN28 | USB-to-UART programmer |
| SN74LVC2G1 | SN74LVC2G241 | Servo half-duplex UART buffer |
| MPU-6050 | MPU-6050 | 6-axis IMU |

---

## Power Architecture

```
VBAT (7.4V 2S) ──┬── TPS5683x → +7V4A (Servos 1-6)
                  ├── TPS5683x → +7V4B (Servos 7-12)
                  ├── TPS5683x → +7V4C (Servos 13-18)
                  └── SGM2212  → +3V3  (MCU, RPi logic)

USB-C (VBUS) ──── CP2102N/AO3401A → +5V (RPi, USB programming)

+3V3 ──── separate +ESP_3V3 (via 0Ω jumper R2, isolatable)
```

---

## Outstanding Items

### Not Yet Placed
- [ ] **BQ25887RGET** — 2S Li-Ion battery charger (I²C, addr 0x6B) — listed in README, not in schematics
- [ ] **S-8252AAZ-M6T1U** — Battery safety IC (overcurrent/overvoltage protection)

### Footprints
- Most component footprints are **not yet assigned** (empty `Footprint` fields). Footprint assignment is a required step before PCB layout.

### ERC
- No ERC run recorded. Run before PCB layout to catch open nets and unconnected pins.

### PCB Layout
- No layout work started (`.kicad_pcb` is placeholder/empty).

---

## Next Steps

1. Add BQ25887 charger circuit to `power_routes.kicad_sch`
2. Add S-8252AAZ battery protection to `power_routes.kicad_sch`
3. Commit `programmer.kicad_sch`
4. Assign footprints to all components
5. Run ERC, resolve all errors
6. Begin PCB layout — board outline and component placement
