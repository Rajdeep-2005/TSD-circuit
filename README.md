
# Thermal Shutdown System for Power MOSFET (PMIC Project)

A fully analog, on-chip **Thermal Shutdown (TSD)** circuit designed for a power MOSFET(BSC009NE2LS5), modeled and verified in **Cadence Virtuoso (gpdk090)**. The system senses junction temperature using a thermal RC ladder derived from real Infineon datasheet data, compares it against a **Bandgap Reference (BGR)**, and uses a **hysteretic comparator** to shut the device down on overheating and safely re-enable it once it cools.

> Project : Thermal Dynamics and Protection — IIIT Bangalore

**Authors:** Rajdeep Alapati (A. Rajdeep, IMT2023592) & Kotyada Parthi / K. Parthiv (IMT2023559)

---

## 1. Motivation

Power MOSFETs and dense CMOS dies heat up due to:

- Switching losses from charging/discharging parasitic capacitances
- Sub-threshold leakage current, worsened by scaling
- High power density and localized hot spots

Left unchecked, this leads to **thermal runaway** (heat → more leakage → more heat), permanent silicon damage, and reduced device lifetime. A thermal shutdown block protects the die by disconnecting it before the junction temperature crosses a safe limit, and re-enabling it once it has cooled — without chattering near the trip point.

## 2. System Architecture

```
                ┌───────────────┐
                │  Bandgap Ref   │   Vref ≈ 1.28 V
                │   (BGR)        │ ─────────────┐
                └───────────────┘               │
                                                 ▼
┌───────────────┐   Vtemp   ┌───────────────────────────┐
│ Thermal RC     │──────────▶│   Comparator + Hysteresis  │── Switchoff
│ Ladder (sensor)│           │   (Shutdown Controller)    │   (control signal)
└───────────────┘           └───────────────────────────┘
        ▲
        │ Pdiss (power dissipated → modeled as current)
```

- **Block 1 — BGR:** Generates a temperature-stable reference voltage (~1.28 V) by summing a CTAT (V_BE) and a PTAT (ΔV_BE) term so that ∂V_BGR/∂T ≈ 0.
- **Block 2 — Thermal Sensor:** A Foster RC ladder (R_th, C_th) converts dissipated power into a CTAT voltage that tracks junction temperature, using the temperature↔voltage, power↔current electrical analogy (τ = R_th·C_th).
- **Block 3 — Comparator/Controller:** Compares V_temp against the reference with dual thresholds (hysteresis) and drives the `Switchoff` control signal.

## 3. Design Details

### 3.1 Thermal Modeling — Foster RC Ladder

Device under protection: **Infineon BSC009NE2LS5** (OptiMOS™5 Power MOSFET, 25 V, N-channel, SuperSO8 package).

Datasheet thermal characteristics used as design inputs:

| Parameter | Symbol | Max | Unit |
|---|---|---|---|
| Thermal resistance, junction–case (bottom) | R_thJC | 1.7 | K/W |
| Thermal resistance, junction–case (top) | R_thJC | 20 | K/W |
| Thermal resistance, device on PCB, 6 cm² cooling area | R_thJA | 50 | K/W |

The transient thermal impedance curve Z_th(t) from the datasheet was fitted to a 3-stage Foster RC network (R_th in series with C_th to ground), giving the thermal RC ladder used as the sensor:

| Stage | R_th | C_th |
|---|---|---|
| 1 (junction-near) | ≈ 0.25 Ω | 160 µF |
| 2 | ≈ 0.56 Ω | 1.45 mF |
| 3 (slow/board-level) | ≈ 0.9 Ω | 11.1 mF |

*(Values as extracted from the Virtuoso schematic — double-check against your final netlist before publishing.)*

### 3.2 Bandgap Reference (BGR)

Built from an NPN BJT pair (Q1, Q2) at an area ratio *n*, PMOS current mirrors to force equal collector currents, and a resistor network to set the PTAT gain *K*:

```
V_BGR = V_BE (CTAT, ≈ −2 mV/°C)  +  K · V_T·ln(n) (PTAT)
```

Simulated DC sweep from −40 °C to 160 °C confirms first-order cancellation: V_BGR stays essentially flat at **≈ 1.28 V** across the full range while the individual CTAT and PTAT branches slope in opposite directions.

### 3.3 Comparator and Hysteresis

A single fixed threshold causes chattering (rapid ON/OFF toggling) near the trip point due to noise. The design instead uses **dual thresholds**:

- **Shutdown trip:** ~150–155 °C
- **Restart trip:** ~100 °C
- **Hysteresis window:** ΔT ≈ 50–55 °C

This keeps the system stable, only re-enabling the device once it has meaningfully cooled.

## 4. Simulation Results

- **BGR DC sweep:** Output stable at ≈1.28 V from −40 °C to 160 °C — confirms temperature compensation.
- **Transient response (1 °C = 1 V mapping):** Junction voltage cycles between the restart and shutdown thresholds while `/Switchonoff` toggles correctly between ON and OFF, confirming correct hysteretic behavior. Example measured cursor points from the waveform: M1 = 100.834 V (≈100.8 °C) at the restart edge, M2 = 149.464 V (≈149.5 °C) at the shutdown edge.

## 5. Tools

- **Cadence Virtuoso** (Schematic Editor + ADE Explorer/XL)
- **gpdk090** process design kit
- DC sweep and transient analyses


## 6. Future Work

- Layout and post-layout extraction to verify thermal RC values against physical die parasitics.
- Process/voltage corner analysis (PVT) for the BGR and comparator thresholds.
- Replacing the comparator with a Schmitt-trigger-style single-stage design to reduce area.

## 7. References

1. Infineon, *BSC009NE2LS5 Datasheet* — OptiMOS™5 Power MOSFET.
2. Jeremy Howes, "Temperature Limits for Power Modules Part-1: Maximum Junction Temperature," *EE Power*, Jan 1, 2016.
3. SemiEngineering (Reela Samuel) — thermal hot-spot distribution reference image.

---

*IIIT Bangalore : Thermal Dynamics and Protection*
