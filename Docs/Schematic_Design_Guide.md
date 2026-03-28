# Balancing Robot -- Unified Single-Board Schematic Design Guide

## System Architecture Overview

All electronics reside on a **single PCB** divided into three logical zones separated by **keep-out areas** to isolate high-power motor noise from sensitive logic. All external connectors use **JST-PH (2.0 mm pitch)**. Motor drivers use the **Pololu DRV8874 carrier board** (#4035), which is soldered onto the main PCB via 0.1" through-hole headers.

```
+===========================================================================+
|                                                                           |
|   MOTOR ZONE A           LOGIC ZONE (center)          MOTOR ZONE B       |
|   (left side)                                          (right side)       |
|                    +---------keep-out---------+                           |
|  +-------------+  :                           :  +-------------+          |
|  | Pololu      |  :   [ICM-20948 IMU]        :  | Pololu      |          |
|  | DRV8874     |  :                           :  | DRV8874     |          |
|  | carrier (A) |  :   [Teensy 4.1]            :  | carrier (B) |          |
|  | (15x18mm)   |  :                           :  | (15x18mm)   |          |
|  +-------------+  :   [PM02 sense resistors]  :  +-------------+          |
|  [nFAULT R]       :                           :  [nFAULT R]              |
|  [IPROPI filter]  +---------keep-out---------+   [IPROPI filter]         |
|  [Motor EMI caps]                                [Motor EMI caps]         |
|                                                                           |
|  [J3 Motor+Enc A]        [J1 PM02 Power]         [J4 Motor+Enc B]       |
|                           [J2 Battery VM]                                 |
|   (M3)               (M3)            (M3)               (M3)             |
+===========================================================================+

  All connections between zones are PCB traces (no cables needed).
  Keep-out areas: no copper pour, no traces (acts as noise barrier).
  Each zone has its own copper pour with GND bridges at keep-out crossings.
  Pololu carriers soldered directly via 0.1" pin headers for vibration resistance.
```

**External connectors (JST-PH 2.0mm only):**

| Connector | Type | Function | Location |
|-----------|------|----------|----------|
| J1 | JST-PH 4-pin | PM02 power input (5V, GND, V_SENSE, I_SENSE) | Bottom center |
| J2 | JST-PH 2-pin | Battery VM input (7.4V raw, for both motor drivers) | Bottom center |
| J3 | JST-PH 6-pin | Motor A + Encoder A (mates with Hiwonder cable) | Left edge |
| J4 | JST-PH 6-pin | Motor B + Encoder B (mates with Hiwonder cable) | Right edge |

---

## 1. Component & Connector List (BOM)

### Single Unified PCB

**Logic Zone (Center)**

| Ref | Component | Description | Qty |
|-----|-----------|-------------|-----|
| U1 | Teensy 4.1 | ARM Cortex-M7 @ 600 MHz, with pin headers | 1 |
| U2 | Adafruit ICM-20948 | 9-DoF IMU breakout (Accel/Gyro/Mag) | 1 |
| J1 | JST-PH 4-pin (S4B-PH-K-S) | PM02 power input | 1 |
| R1, R2 | 10 kOhm 1/4W 1% | Voltage divider for V_SENSE | 2 |
| R3 | 10 kOhm 1/4W | Current sense load resistor | 1 |
| C1 | 100 nF ceramic X7R | Decoupling on Teensy VIN | 1 |
| C2 | 10 uF ceramic X5R | Bulk cap on Teensy VIN | 1 |

**Motor Zone A (Left) -- Identical to Motor Zone B**

The Pololu DRV8874 carrier (15.2 x 17.8 mm) already includes: DRV8874 IC, VM bypass cap, charge pump caps (storage + flying), reverse-voltage protection MOSFET, IMODE 20k pulldown, VREF 10k pullup to SLEEP, and RIPROPI 2.49k to GND (CS output at ~1.1 V/A). The only external components needed per zone are:

| Ref (Zone A / Zone B) | Component | Description | Qty each |
|------------------------|-----------|-------------|----------|
| U3a / U3b | Pololu DRV8874 carrier (#4035) | Motor driver breakout, 0.1" headers | 1 |
| J3 / J4 | JST-PH 6-pin (S6B-PH-K-S) | Motor + encoder connector | 1 |
| C4a / C4b | 100 uF 16V electrolytic | VM bulk cap (recommended for wire runs to J2) | 1 |
| C7a / C7b | 100 nF ceramic | Motor terminal EMI cap (OUT1/M1) | 1 |
| C8a / C8b | 10 nF ceramic | Motor terminal EMI cap (OUT2/M2) | 1 |
| R4a / R4b | 10 kOhm 1/4W | FAULT pullup to 3.3V (not on Pololu board) | 1 |
| R6a / R6b | 10 kOhm 1/4W | CS low-pass filter resistor (optional) | 1 |
| C9a / C9b | 10 nF ceramic | CS low-pass filter cap (optional) | 1 |

**Shared / Board-Level**

| Ref | Component | Description | Qty |
|-----|-----------|-------------|-----|
| J2 | JST-PH 2-pin (S2B-PH-K-S) | Battery VM input (shared, splits to both zones) | 1 |
| FB1, FB2 | Ferrite bead 600 Ohm @ 100 MHz | 3.3V isolation into each motor zone (optional) | 2 |
| -- | 0.1" pin headers (13-pin) | For mounting each Pololu carrier to main PCB | 2 sets |
| -- | M3 mounting holes (3.2 mm) | Board corners, symmetric | 4 |

> **What the Pololu carrier eliminates from the BOM (per zone):** VM bypass cap (100nF),
> charge pump storage cap (100nF), charge pump flying cap (22nF), RIPROPI sense resistor
> (2.49k on-board), IMODE pulldown resistor (20k on-board), VREF pullup resistor (10k
> on-board to SLEEP), and the bare DRV8874 HTSSOP-16 IC itself. That is 7 SMD components
> per zone (14 total) replaced by one drop-in module.

---

## 2. Complete Pin Mapping Table

### Teensy 4.1 Master Pin Assignment

| Teensy Pin | Arduino Name | Function | Connected To | Notes |
|------------|-------------|----------|--------------|-------|
| **VIN** | VIN | 5V power input | J1 Pin 1 (PM02 5V) | 3.6-5.5V; PM02 outputs 5.2V |
| **GND** | GND | System ground | Logic zone ground pour | Star-ground point connects all zones |
| **3V3** | 3.3V | Regulated 3.3V out | U2, encoders, VREF, nFAULT pullups | 250 mA max (see power budget) |
| | | | | |
| | | **--- IMU (SPI0) ---** | | |
| **13** | D13 / SCK0 | SPI0 SCK | U2 SCL (ICM-20948 SCK) | SPI clock, up to 7 MHz |
| **11** | D11 / MOSI0 | SPI0 MOSI | U2 SDA (ICM-20948 SDI) | SPI data to IMU |
| **12** | D12 / MISO0 | SPI0 MISO | U2 SDO (ICM-20948 SDO) | SPI data from IMU |
| **10** | D10 / CS0 | SPI0 CS | U2 CS (ICM-20948 nCS) | Chip select, active low |
| **17** | A3 | IMU Interrupt | U2 INT | Data-ready, INPUT_PULLUP |
| | | | | |
| | | **--- PM02 Power ---** | | |
| **14** | A0 | Analog input | J1 Pin 3 via R1/R2 divider | Battery voltage sense |
| **15** | A1 | Analog input | J1 Pin 4 via R3 | Battery current sense |
| | | | | |
| | | **--- Motor A (Left Zone, Pololu A) ---** | | |
| **2** | D2 | PWM output | Pololu A: EN/IN1 | FlexPWM4_PWMA02, speed |
| **3** | D3 | Digital output | Pololu A: PH/IN2 | Direction |
| **29** | D29 | Digital output | Pololu A: SLEEP | Motor A enable (active HIGH) |
| **0** | D0 / RX1 | Digital input (IRQ) | J3 Pin 3 -> encoder A phase A | Quadrature channel A |
| **1** | D1 / TX1 | Digital input (IRQ) | J3 Pin 4 -> encoder A phase B | Quadrature channel B |
| **20** | A6 | Analog input | Pololu A: CS via R6a/C9a filter | Motor A current (~1.1 V/A) |
| **33** | D33 | Digital input | Pololu A: FAULT | Fault, active low, ext. pullup |
| | | | | |
| | | **--- Motor B (Right Zone, Pololu B) ---** | | |
| **5** | D5 | PWM output | Pololu B: EN/IN1 | FlexPWM2_PWMA01, speed |
| **6** | D6 | Digital output | Pololu B: PH/IN2 | Direction |
| **30** | D30 | Digital output | Pololu B: SLEEP | Motor B enable (active HIGH) |
| **31** | D31 | Digital input (IRQ) | J4 Pin 3 -> encoder B phase A | Quadrature channel A |
| **32** | D32 | Digital input (IRQ) | J4 Pin 4 -> encoder B phase B | Quadrature channel B |
| **21** | A7 | Analog input | Pololu B: CS via R6b/C9b filter | Motor B current (~1.1 V/A) |
| **34** | D34 | Digital input | Pololu B: FAULT | Fault, active low, ext. pullup |

### ICM-20948 Breakout (U2) Pin Connections (SPI Mode)

| IMU Pin | Connected To | Notes |
|---------|-------------|-------|
| VIN | Teensy 3V3 | Powers on-board 1.8V regulator and level shifters |
| GND | Logic zone GND | |
| SCL | Teensy Pin 13 (SCK0) | SPI clock (labeled SCL on breakout, serves as SCK in SPI mode) |
| SDA | Teensy Pin 11 (MOSI0) | SPI data in (labeled SDA on breakout, serves as SDI/MOSI) |
| SDO | Teensy Pin 12 (MISO0) | SPI data out (MISO). Active when CS is low |
| CS | Teensy Pin 10 (CS0) | **Must be driven LOW to select SPI mode** |
| INT | Teensy Pin 17 | Active-low open-drain; use INPUT_PULLUP in firmware |

> **SPI Mode Selection:** On the Adafruit ICM-20948 breakout, pulling CS LOW selects SPI mode
> (the chip latches protocol at power-up based on CS state). The breakout's on-board I2C pullups
> on SCL/SDA remain present but do not interfere with SPI operation -- they simply pull the
> lines high when idle, which is normal SPI idle state (CPOL=0).

### PM02 Power Module (J1) -- JST-PH 4-Pin

| J1 Pin | Signal | Connected To | Notes |
|--------|--------|-------------|-------|
| 1 | +5V (regulated) | Teensy VIN | PM02 regulated 5.2V output |
| 2 | GND | Logic zone GND | |
| 3 | V_SENSE | R1/R2 divider -> Teensy A0 (Pin 14) | Battery voltage analog signal |
| 4 | I_SENSE | R3 -> Teensy A1 (Pin 15) | Battery current analog signal |

> **IMPORTANT NOTE:** The documentation in the Docs folder is for the **PM02D** variant,
> which uses **I2C** (SCL/SDA on pins 3/4) for voltage/current monitoring instead of analog
> outputs. If you are using the PM02D:
> - Connect PM02D SCL to Teensy Pin 19 (SCL0 / Wire) and SDA to Teensy Pin 18 (SDA0 / Wire).
>   Since the IMU now uses SPI, the I2C0 bus is free for PM02D.
> - Pins A0/A1 become unused for PM02 sensing
> - The I2C address for the PM02D INA226 is typically 0x40

### Battery Input (J2) -- JST-PH 2-Pin

| J2 Pin | Signal | Connected To | Notes |
|--------|--------|-------------|-------|
| 1 | VM+ | Motor zone A and B VM rails (via copper pour) | 7.4V raw battery, 2.8A peak |
| 2 | GND | Motor zone ground pours | |

### Motor + Encoder Connectors (J3, J4) -- JST-PH 6-Pin

These directly mate with the Hiwonder JGA27-310R20 motor cable (PH2.0 6-pin).

| Pin | Hiwonder Wire | Signal | Connected To |
|-----|---------------|--------|-------------|
| 1 | Wire 1 | M2 (motor -) | DRV8874 OUT2 (Pin 10) via EMI cap to GND |
| 2 | Wire 2 | V (encoder power) | 3.3V rail |
| 3 | Wire 3 | A (encoder phase A) | Teensy digital input (Pin 0 / Pin 31) |
| 4 | Wire 4 | B (encoder phase B) | Teensy digital input (Pin 1 / Pin 32) |
| 5 | Wire 5 | G (encoder ground) | GND |
| 6 | Wire 6 | M1 (motor +) | DRV8874 OUT1 (Pin 8) via EMI cap to GND |

---

## 3. Netlist Wiring Guide -- Logic Zone (Center)

### 3.1 Power Distribution

```
PM02 5V (J1 Pin 1) ----+---- C2 (10uF) ----+---- GND (logic zone)
                        |                    |
                        +---- C1 (100nF) ----+
                        |
                        +---- Teensy VIN

Teensy 3V3 output -----+---- ICM-20948 VIN (U2)
                        |
                        +---- Encoder A power (J3 Pin 2)
                        |
                        +---- Encoder B power (J4 Pin 2)
                        |
                        +---- U3a VREF (Pin 5) [via trace to motor zone A]
                        |
                        +---- U3b VREF (Pin 5) [via trace to motor zone B]
                        |
                        +---- R4a, R4b nFAULT pullups
```

**Power budget (3.3V rail):**
- ICM-20948: ~3.1 mA (9-axis mode)
- Encoders x2: ~10 mA each
- VREF x2: negligible (high impedance)
- nFAULT pullups x2: ~0.3 mA each
- Total: ~33.7 mA (well within Teensy 3.3V 250 mA limit)

### 3.2 SPI Bus (SPI0)

```
Teensy Pin 13 (SCK0)  ---------- ICM-20948 SCL  (SPI Clock)
Teensy Pin 11 (MOSI0) ---------- ICM-20948 SDA  (SPI Data In / SDI)
Teensy Pin 12 (MISO0) ---------- ICM-20948 SDO  (SPI Data Out)
Teensy Pin 10 (CS0)   ---------- ICM-20948 CS   (Chip Select, active low)
```

- **SPI clock speed:** Up to **7 MHz** supported by the ICM-20948 (vs. 400 kHz I2C max). ~17.5x faster bus throughput.
- **SPI Mode:** Mode 0 (CPOL=0, CPHA=0) or Mode 3 (CPOL=1, CPHA=1) per ICM-20948 datasheet.
- The Adafruit breakout has on-board level shifters that work in both I2C and SPI modes.
- The breakout's 10K I2C pullups on SCL/SDA do not interfere with SPI (they pull idle-high, matching CPOL=0).
- **CS must be held LOW during all SPI transactions** and driven HIGH between transactions.

### 3.3 IMU Interrupt

```
ICM-20948 INT ---- Teensy Pin 17 (A3)
```

- Configure as `INPUT_PULLUP` in firmware (INT is open-drain active-low).
- Use for data-ready interrupt to trigger IMU reads at precise intervals.

### 3.4 PM02 Analog Sensing (Original PM02 with Analog Outputs)

#### Battery Voltage Sense (V_SENSE)

```
J1 Pin 3 (V_SENSE) ---- R1 (10K) ----+---- Teensy Pin 14 (A0)
                                       |
                                       R2 (10K)
                                       |
                                       GND
```

This 2:1 divider provides safety margin. Calibrate in firmware:
```
V_battery = analogRead(A0) * (3.3 / 4095.0) * CALIBRATION_FACTOR;
```

If the PM02 V_SENSE is already scaled for 3.3V ADC, connect directly to A0 and add only a 100 nF filter cap to GND.

#### Battery Current Sense (I_SENSE)

```
J1 Pin 4 (I_SENSE) ---- Teensy Pin 15 (A1)
                    |
                    +---- 100nF cap ---- GND  (noise filter)
```

### 3.5 Motor Control Signals (PCB traces to motor zones)

All signals are direct PCB traces crossing the keep-out area. No connectors needed.

```
Motor A (traces to left zone):            Motor B (traces to right zone):
  Teensy Pin 2  (PWM_A)   -> U3a EN/IN1     Teensy Pin 5  (PWM_B)   -> U3b EN/IN1
  Teensy Pin 3  (DIR_A)   -> U3a PH/IN2     Teensy Pin 6  (DIR_B)   -> U3b PH/IN2
  Teensy Pin 29 (nSLEEP_A)-> U3a nSLEEP     Teensy Pin 30 (nSLEEP_B)-> U3b nSLEEP
  Teensy Pin 20 (IPROPI_A)<- U3a filter      Teensy Pin 21 (IPROPI_B)<- U3b filter
  Teensy Pin 33 (nFAULT_A)<- U3a nFAULT     Teensy Pin 34 (nFAULT_B)<- U3b nFAULT
  Teensy Pin 0  (ENC_A_A) <- J3 Pin 3        Teensy Pin 31 (ENC_B_A) <- J4 Pin 3
  Teensy Pin 1  (ENC_A_B) <- J3 Pin 4        Teensy Pin 32 (ENC_B_B) <- J4 Pin 4
```

- **PWM frequency:** 20 kHz (above audible range, within DRV8874 capability).
- **PWM resolution:** 10-bit (0-1023) at 20 kHz on Teensy 4.1 FlexPWM.
- All signals are 3.3V logic. DRV8874 VIH = 1.5V, so 3.3V is fully compatible.
- **IPROPI traces** cross the keep-out as narrow, ground-guarded traces routed to Teensy ADC pins (A6, A7). Place 100 nF filter caps within 5 mm of the Teensy ADC pins.
- **nFAULT signals** are pulled up by R4a/R4b (10K to 3.3V) on the motor zones. Read as digital inputs (LOW = fault).

### 3.6 Encoder Notes

- Encoder signals enter via J3/J4 in the motor zones and route to the Teensy in the logic zone.
- Teensy 4.1 has hardware quadrature encoder support via the ENC module. Pins 0/1 map to Encoder1.
- Pins 31/32 can use the Encoder library with interrupts (all Teensy 4.1 digital pins support interrupts).
- The Hiwonder encoders have built-in pull-up shaping circuits; no external pullups needed.
- 13 PPR x 20:1 gear ratio x 4 (quadrature) = **1040 counts per output revolution**.

### 3.7 Traces Crossing Keep-Out Zones

The following traces must cross from the logic zone into each motor zone. These are the **only** connections bridging the keep-out areas:

**Per motor zone (7 signal traces + 2 power):**

| Trace | Type | Width | Notes |
|-------|------|-------|-------|
| PWM (EN) | Digital out, 20 kHz | 0.3 mm (12 mil) | Fast-switching; keep short |
| DIR (PH) | Digital out, DC | 0.25 mm (10 mil) | Low frequency |
| nSLEEP | Digital out, DC | 0.25 mm (10 mil) | Low frequency |
| ENC_A | Digital in | 0.25 mm (10 mil) | From motor connector |
| ENC_B | Digital in | 0.25 mm (10 mil) | From motor connector |
| IPROPI | Analog in | 0.3 mm (12 mil) | **Ground-guarded on both sides** |
| nFAULT | Digital in, open-drain | 0.25 mm (10 mil) | Pulled up in motor zone |
| 3.3V | Power | 0.5 mm (20 mil) | For encoder, VREF, pullups |
| GND bridge | Signal return path | 1.0 mm (40 mil) | **Adjacent to signal group; one per zone** |

> **Critical rule:** Route ALL zone-crossing traces as a **tight, parallel group** through
> the narrowest possible gap in the keep-out area. Do not fan them out across the keep-out.
> The GND bridge trace must be **immediately adjacent** to the signal group -- it is the
> local return path for every signal in the group. Without it, signal return currents would
> have to travel off-board through the PM02 cables (see Section 5.1 for full explanation).

---

## 4. Netlist Wiring Guide -- Motor Zones (Left & Right)

Both motor zones are **mirror images** of each other. The schematic is identical; only the Teensy pin assignments differ (see pin mapping table).

### 4.1 Pololu DRV8874 Carrier -- What's Already On-Board

Before wiring, understand what the Pololu carrier provides so you don't duplicate components:

| Feature | On Pololu Carrier | External (on main PCB) |
|---------|-------------------|----------------------|
| DRV8874 IC | Yes (pre-soldered HTSSOP) | -- |
| VM bypass cap (100nF) | Yes | -- |
| Charge pump storage cap (100nF) | Yes | -- |
| Charge pump flying cap (22nF) | Yes | -- |
| Reverse-voltage protection MOSFET | Yes (between VIN and VM) | -- |
| RIPROPI / CS resistor | Yes (2.49 kOhm to GND) | -- |
| IMODE pulldown | Yes (20 kOhm to GND) | -- |
| VREF pullup | Yes (10 kOhm to SLEEP pin) | -- |
| VM bulk electrolytic cap | No | C4 (100uF 16V) recommended |
| FAULT pullup resistor | No | R4 (10k to 3.3V) required |
| CS low-pass filter | No | R6 + C9 (optional, recommended) |
| Motor EMI caps | No | C7 + C8 at JST-PH connector |

### 4.2 Pololu Carrier Pinout (13 pins on 0.1" headers)

```
  Motor/Power side:          Logic/Control side:
  +--------------+           +--------------+
  | VIN          |           | EN/IN1       |  <- PWM speed (from Teensy)
  | GND          |           | PH/IN2       |  <- Direction (from Teensy)
  | VM           |           | PMODE        |  <- Tie LOW for PH/EN mode
  | OUT1         |           | SLEEP        |  <- Enable (from Teensy, active HIGH)
  | OUT2         |           | VREF         |  <- (leave floating, pulled to SLEEP on-board)
  +--------------+           | IMODE        |  <- (leave floating, pulled LOW on-board)
                             | FAULT        |  <- Fault output (add pullup)
                             | CS           |  <- Current sense (~1.1V/A)
                             +--------------+
```

### 4.3 Battery Power Input

```
J2 Pin 1 (VM+, 7.4V) ----[copper pour]----+---- Pololu A: VIN  [left zone]
                                            |
                                            +---- Pololu B: VIN  [right zone]

J2 Pin 2 (GND) ----[copper pour]----+---- Motor zone A ground pour
                                     |         (connects to Pololu A: GND)
                                     |
                                     +---- Motor zone B ground pour
                                               (connects to Pololu B: GND)
```

- J2 is placed at the board bottom-center, between or adjacent to the motor zones. Heavy copper pours carry VM to each motor zone.
- **J2 GND connects directly to the motor zone ground pours** (not to the logic pour). This ensures high-current motor return paths stay within the motor zones and exit through J2 back to the PM02 battery ground.
- J1 GND connects to the **logic zone ground pour**. The PM02 closes the ground loop internally.
- Trace/pour width for VM: **1.5 mm (60 mil) minimum** or copper pour (2.8A peak combined).
- Place C4 (100uF bulk cap) between VIN and GND on the main PCB, within 10mm of each Pololu carrier's VIN pin.

### 4.4 Pololu Carrier Wiring (per motor zone)

#### Logic Inputs (from Teensy via zone-crossing traces)

```
Teensy PWM pin   ---- Pololu: EN/IN1     [PWM speed control]
Teensy DIR pin   ---- Pololu: PH/IN2     [Direction control]
Teensy SLEEP pin ---- Pololu: SLEEP      [Enable, active HIGH]
```

> **SLEEP replaces nSLEEP in the pin naming.** The function is identical -- drive HIGH to
> enable the driver, LOW for sleep mode. The Pololu board defaults to sleep (SLEEP pin
> has internal 100k pulldown on the DRV8874 IC). You must actively drive it HIGH.

#### Mode Configuration

```
Pololu: PMODE ---- GND (motor zone pour)    [PH/EN mode select]
Pololu: IMODE ---- leave unconnected        [20k pulldown on-board = cycle-by-cycle + auto retry]
Pololu: VREF  ---- leave unconnected        [10k pullup to SLEEP on-board]
```

- **PMODE must be tied LOW before SLEEP goes HIGH.** PMODE is latched on the rising edge of SLEEP. If left floating (default), the driver enters independent half-bridge mode (not what we want).
- IMODE is already pulled LOW (20k on-board): cycle-by-cycle current chopping, automatic retry OCP.
- VREF is pulled to SLEEP voltage via 10k on-board. With SLEEP driven at 3.3V from Teensy:
  - VREF ≈ 3.3V
  - ITRIP = VREF / (RIPROPI x AIPROPI) = 3.3V / (2490 Ohm x 455 uA/A) = **2.91A**
  - This is well above the 1.4A stall current -- provides generous headroom.

#### Current Sensing (CS Pin)

```
Pololu: CS ----+---- R6 (10kOhm) ----+---- [trace to Teensy ADC]
               |                      |
               |                      C9 (10nF) ---- GND (motor zone)
               |
               (2.49k to GND already on Pololu board)
```

- The Pololu carrier has a 2.49 kOhm resistor from IPROPI to GND, giving **~1.1 V/A** at the CS pin.
- At rated current (0.65A): V_CS = **0.72V**.
- At stall current (1.4A): V_CS = **1.54V** (well within Teensy 3.3V ADC range).
- R6 + C9 form an optional low-pass filter (fc ~1.6 kHz) to suppress PWM switching noise before the signal crosses the keep-out zone to the Teensy ADC.
- If you don't need current sensing, leave CS unconnected.

#### Fault Output

```
Pololu: FAULT ----+---- R4 (10kOhm) ---- 3.3V (from logic zone)
                  |
                  +---- [trace to Teensy digital input]
```

- FAULT is open-drain, active-low. **No pullup on the Pololu board** -- R4 is required.
- Goes LOW on fault: overcurrent (OCP), undervoltage (UVLO), or thermal shutdown (TSD).

#### Motor Output (to JST-PH connector)

```
Pololu: OUT1 ----+---- J3/J4 Pin 6 (M1, motor +)
                 |
                 C7 (100nF) ---- GND (motor zone)

Pololu: OUT2 ----+---- J3/J4 Pin 1 (M2, motor -)
                 |
                 C8 (10nF) ---- GND (motor zone)
```

> **Motor terminal EMI caps:** C7 and C8 suppress brush commutation noise at the source.
> Place directly at the JST-PH motor connector pins, not near the Pololu carrier.

#### Pololu: VM Pin (unused)

The VM pin provides access to the motor supply *after* the on-board reverse-voltage protection MOSFET. **Leave VM unconnected** unless you need reverse-protected power for other components.

### 4.5 Encoder Pass-Through (per motor zone)

```
J3/J4 Pin 2 (V, encoder power) ---- 3.3V (from logic zone)
J3/J4 Pin 3 (A, encoder phase A) ---- [trace to Teensy digital input]
J3/J4 Pin 4 (B, encoder phase B) ---- [trace to Teensy digital input]
J3/J4 Pin 5 (G, encoder ground) ---- GND (motor zone)
```

### 4.4 DRV8874 PH/EN Mode Truth Table (Reference)

| nSLEEP | EN (IN1) | PH (IN2) | OUT1 | OUT2 | Motor Action |
|--------|----------|----------|------|------|-------------|
| 0 | X | X | Hi-Z | Hi-Z | Sleep (low power) |
| 1 | 0 | X | LOW | LOW | Brake (slow decay) |
| 1 | 1 | 1 | HIGH | LOW | Forward |
| 1 | 1 | 0 | LOW | HIGH | Reverse |

- Apply PWM to EN for speed control. During PWM LOW phase, both outputs go LOW (brake/slow decay).
- Set PH HIGH or LOW to select direction.
- Startup sequence: Set nSLEEP HIGH, wait 1 ms (tWAKE), then begin PWM.

---

## 5. Critical PCB Layout Rules

### 5.0 Self-Balancing Robot: Mechanical & Layout Constraints

A self-balancing robot is a real-time inverted pendulum. The control loop reads the IMU, computes a PID/LQR correction, and drives the motors -- typically at **200-1000 Hz**. This imposes strict requirements on the physical PCB design.

#### IMU Placement (HIGHEST PRIORITY)

The ICM-20948 placement is the single most critical layout decision.

```
                        PITCH AXIS (balancing axis)
                             |
              Motor A        |         Motor B
              Zone     +-----+-----+   Zone
                       |     |     |
                       | [ICM-20948]|  <-- Place ON or as close as
                       |     |     |      possible to the pitch axis
                       | [Teensy]  |
                       |           |
                       +-----------+
                             |
                      Wheel axle line
```

- **Place the IMU as close to the robot's center of rotation (wheel axle) as possible.** Any offset from the axle introduces centripetal acceleration artifacts into the accelerometer reading during rotation, corrupting tilt angle estimation.
- Ideal placement: directly above the wheel axle, centered left-to-right on the board.
- If the IMU cannot be at the axle, place it on the **vertical line through the axle** (directly above it). Vertical offset introduces less error than horizontal offset.
- **Maximum acceptable offset from axle center:** < 30 mm.
- **Mount the IMU flat** -- the breakout board must be parallel to the PCB with zero flex.

#### IMU Orientation

- Align the ICM-20948's **X-axis with the robot's forward/backward direction** (pitch axis measurement) and **Y-axis with the wheel axle** (left-right).
- The Adafruit breakout silkscreen shows axis orientation. Verify this matches your chassis frame before finalizing placement.
- Document the chosen orientation in firmware comments -- getting the axis/sign wrong is the #1 debugging time sink in balancing robots.

#### Vibration Isolation (CRITICAL)

DC motors with gearboxes (like the Hiwonder JGA27-310R20 with its 20:1 metal gear train) generate significant mechanical vibration at the gear mesh frequency and motor RPM harmonics. This vibration directly couples into the IMU accelerometer, corrupting tilt estimation.

**PCB-Level Mitigations:**
- Mount the board to the chassis using **M3 standoffs with silicone or rubber grommets** (e.g., 3mm ID rubber washers between the standoff and the PCB). This provides mechanical low-pass filtering.
- Alternatively, use **soft-mount standoffs** (e.g., Sorbothane pads or silicone gel standoffs) with a resonant frequency well below the gear mesh frequency.
- **Do NOT use only the JST-PH connectors as mechanical support** -- the board must be independently fastened to the frame.

**Firmware-Level Mitigations (noted here for completeness):**
- Configure the ICM-20948's internal digital low-pass filter (DLPF) to filter above 50-100 Hz.
- Use a complementary filter or Kalman/Madgwick filter that fuses gyro + accel to reject high-frequency vibration in the accelerometer.
- The gyroscope is far less sensitive to linear vibration than the accelerometer -- lean on gyro data for short-term tilt and accel for long-term drift correction.

#### Center of Gravity

- **Keep the center of gravity HIGH.** A higher CoG makes the inverted pendulum slower to fall, giving the controller more time to react. Mounting the battery high is beneficial.
- The battery (heaviest component) should be mounted **above the wheel axle**, ideally adjustable in height to tune the balance point during testing.
- The board itself should be **symmetrical left-to-right** about the robot centerline. The zoned layout (Motor A | Logic | Motor B) naturally achieves this.
- **Keep board dimensions compact.** A wider board increases the moment of inertia about the pitch axis.

#### Board Mounting Holes

- Place **4 mounting holes** (M3, 3.2 mm diameter) at the corners.
- Position mounting holes symmetrically about the board centerline.
- Keep mounting holes at least **3 mm from any copper trace** and **5 mm from the IMU**.

### 5.1 Zoned Ground Plane Strategy (CRITICAL)

This is the key layout technique for the single-board design. Instead of one continuous ground plane, use **three separate copper pours** connected by **controlled GND bridge traces** at the keep-out crossing points.

#### Why do we need on-board GND bridges if the PM02 already connects everything?

The PM02 module is the system-level power hub. Internally, its regulated 5V output and raw battery pass-through share a common ground. So electrically, all grounds *are* already connected -- through the PM02, off-board, via cables.

**But that off-board path is the problem.** When Teensy Pin 2 sends a 20 kHz PWM signal to the DRV8874, the DRV8874 interprets that voltage *relative to its own local ground*. The return current for that signal must flow from DRV8874 GND back to Teensy GND. Without an on-board GND bridge, that return current must travel:

```
  DRV8874 GND -> J2 cable -> PM02 GND (inside module) -> J1 cable -> Teensy GND
  (round-trip through ~300mm of cable = large loop antenna, high inductance)
```

At 20 kHz PWM edges (with fast rise times), this long loop radiates EMI and introduces ground bounce that corrupts the voltage reference for both the DRV8874 logic inputs and the IMU. The on-board GND bridge provides a **short, low-impedance return path** (~10-20 mm of copper trace) right next to the signal traces:

```
  DRV8874 GND -> [GND bridge, 10mm] -> Teensy GND  (clean, tight loop)
```

**The PM02 handles DC power distribution. The on-board GND bridges handle signal integrity.**

#### Ground Architecture Diagram

```
                         PM02 MODULE (off-board)
                        /                   \
                  J1 cable               J2 cable
                  (5V + GND)             (VM + GND)
                      |                      |
+=====================|======================|========================+
|                     |                      |                        |
|  MOTOR GND     +----+--(keep-out)--+-------+----+    MOTOR GND     |
|  POUR A        |    |              |       |    |    POUR B         |
|                |    |  LOGIC GND   |       |    |                   |
|  [DRV8874 A]   |    |  POUR        |       |    |  [DRV8874 B]     |
|  [PGND,GND,   |    |              |       |    |  [PGND,GND,      |
|   therm pad]   |    |  [Teensy]    |       |    |   therm pad]     |
|                |    |  [IMU]       |       |    |                   |
|                |    |              |       |    |                   |
|       GND bridge    |              |    GND bridge                  |
|       (at keep-out  |   [J1]       |    (at keep-out                |
|        crossing) ===+==============+===  crossing)                  |
|                     |    [J2]      |                                |
|                     |              |                                |
+=====================|==============|================================+

  === GND bridge traces (1.0mm wide, at keep-out crossing points)
  These provide the LOCAL return path for signals crossing zones.
  The PM02 cables provide the SYSTEM-LEVEL power return path.
```

#### Rules

1. **Three separate copper pours** on the ground plane layer: Logic GND (center), Motor A GND (left), Motor B GND (right).
2. **Keep-out areas** (2-3 mm wide gap, no copper) separate the motor pours from the logic pour.
3. **GND bridge traces** (1.0 mm / 40 mil wide) connect each motor pour to the logic pour **at the same point where signal traces cross the keep-out**. This ensures the signal and its return current travel together as a tight pair.
4. **One GND bridge per zone crossing** -- do not add extra bridges elsewhere. Multiple connections between pours would allow motor return currents to find alternate paths through the logic ground, defeating the isolation.
5. **Motor high-current return paths** stay within their local motor pour: DRV8874 PGND -> motor zone pour -> J2 GND cable -> PM02. These currents never cross the GND bridge because J2 connects directly to the motor pours.
6. **Logic return paths** stay within the logic pour: Teensy GND -> logic pour -> J1 GND cable -> PM02.
7. **Signal return currents** (small, < 1 mA) flow through the GND bridge. This is fine -- the bridge is right next to the signal trace, forming a tight loop.

#### In KiCad: How to Implement

- Create three ground net zones (or use zone priority/keep-out rules):
  - All three pours are the **same GND net** -- KiCad just pours copper in each zone separately because of the keep-out areas.
  - The keep-out areas prevent the pour from being continuous, naturally creating the gaps.
  - The GND bridge trace is a manually routed trace on the ground plane layer connecting the pours through the keep-out gap.
- Alternatively, use **zone fill rules** with exclusion regions and route the bridge as a trace segment.

### 5.2 Logic Zone Layout (Center)

#### SPI Routing
- Route all four SPI signals (SCK, MOSI, MISO, CS) as a **tightly grouped bus** with matched lengths.
- Keep traces **under 50 mm** total length from Teensy to IMU.
- Place the ICM-20948 breakout as close to the Teensy as physically possible.
- Do not route SPI traces near PWM or motor control signal traces.
- Add a **ground trace or ground pour** between SPI signals and any adjacent noisy traces.
- Trace width: 0.25 mm (10 mil). Route SCK with minimal stubs.
- **Do not route any traces under the IMU footprint on the ground plane layer.** Keep solid, unbroken copper under the IMU.

#### Analog Sensing (A0, A1)
- Route V_SENSE and I_SENSE traces **away from PWM lines** and the SPI bus.
- Use a **ground guard ring** or ground traces flanking the analog signal traces.
- Add 100 nF filtering capacitors at the Teensy ADC input pins, placed within 5 mm of the pin.
- Trace width: 0.25 mm (10 mil).

#### IPROPI Analog Return Traces
- These cross from motor zones into the logic zone. They are the most noise-sensitive traces on the board.
- **Ground-guard** each IPROPI trace (flank with GND traces on both sides as it crosses the keep-out).
- Place 100 nF filter caps on IPROPI_A (A6, Pin 20) and IPROPI_B (A7, Pin 21) within **5 mm** of the Teensy ADC pins.
- Route away from PWM and SPI traces.

#### Power Traces
- **5V from J1 to Teensy VIN:** 0.5 mm (20 mil) minimum trace width.
- **3.3V from Teensy to IMU, encoders, VREF:** 0.3 mm (12 mil) minimum.
- Place decoupling caps (C1, C2) within 5 mm of Teensy VIN pin.

#### Component Placement Priority
1. **IMU first** -- place at the target location (above wheel axle center) before anything else.
2. **Teensy second** -- place adjacent to IMU, minimizing SPI trace length.
3. **PM02 sense resistors** -- place near Teensy ADC pins.
4. **J1 connector** -- bottom edge of logic zone.

### 5.3 Motor Zone Layout (Left & Right, mirrored)

#### Pololu Carrier Mounting
- Create a **through-hole footprint** for the Pololu carrier's 13 pins (0.1" / 2.54 mm pitch).
  - Motor/power side: 5 pins in a row (VIN, GND, VM, OUT1, OUT2).
  - Logic/control side: 8 pins in a row (EN/IN1, PH/IN2, PMODE, SLEEP, VREF, IMODE, FAULT, CS).
  - The two rows are spaced 0.6" (15.24 mm) apart (the board width).
- **Solder the carrier directly** using pin headers (not sockets) for vibration resistance in the balancing robot.
- The Pololu board is 15.2 x 17.8 mm -- plan the motor zone to be approximately **25 x 30 mm** to fit the carrier plus external components (C4, R4, R6, C9, C7, C8).

#### Thermal Notes
- The Pololu carrier handles its own thermal management via its internal ground plane and exposed copper.
- The main PCB's motor zone ground pour beneath the carrier provides additional heat spreading.
- At 1.4A stall with 200 mOhm total RDS_on: 0.39W dissipation, ~14 C rise. Comfortable margin.

#### External Component Placement
- **C4 (100 uF bulk):** Place within **10 mm** of the Pololu VIN pin, on the main PCB.
- **C7/C8 (motor EMI caps):** Place directly at JST-PH motor connector pins (not near the carrier).
- **R4 (FAULT pullup):** Place near the Pololu FAULT pin, pullup to 3.3V.
- **R6/C9 (CS filter):** Place near the Pololu CS pin, before the trace crosses the keep-out.

#### High-Current Trace Widths
| Trace | Current | Minimum Width (1 oz Cu) | Recommended |
|-------|---------|------------------------|-------------|
| VM pour (from J2 to Pololu VIN) | 1.4A stall per motor | 1.0 mm (40 mil) | Copper pour |
| Pololu OUT1/OUT2 to J3/J4 | 1.4A stall | 0.75 mm (30 mil) | 1.0 mm (40 mil) |
| Motor zone GND return | 1.4A stall | 1.0 mm (40 mil) | Ground pour |
| Logic signals crossing keep-out | < 1 mA | 0.25 mm (10 mil) | 0.3 mm (12 mil) |

#### Connector Placement
- **J3 (Motor A + Encoder A):** Left board edge, facing outward toward the left motor.
- **J4 (Motor B + Encoder B):** Right board edge, facing outward toward the right motor.
- **J2 (Battery VM):** Bottom-center, between the two motor zones.

### 5.4 Cable Guidelines

With the single-board design, the only external cables are:

| Cable | From | To | Wire Gauge | Max Length |
|-------|------|-----|-----------|-----------|
| PM02 power | PM02 module | J1 (JST-PH 4-pin) | 22-24 AWG | < 200 mm |
| Battery VM | PM02 XT60 output | J2 (JST-PH 2-pin) | 18-20 AWG | < 150 mm |
| Motor A + Encoder A | Hiwonder motor | J3 (JST-PH 6-pin) | Stock cable | As short as possible |
| Motor B + Encoder B | Hiwonder motor | J4 (JST-PH 6-pin) | Stock cable | As short as possible |

- **Strain-relieve all cables.** A balancing robot oscillates continuously during operation.
- Route motor cables **symmetrically** left-to-right to avoid asymmetric mass.
- The stock Hiwonder PH2.0 cable plugs directly into J3/J4 with no adapters needed.

### 5.5 EMI and Noise Mitigation

- **Motor EMI caps** (C7/C8 per zone) suppress brush commutation noise at the source.
- **Optional ferrite beads** (FB1, FB2): place in series with the 3.3V trace as it crosses each keep-out area. This prevents high-frequency motor noise from coupling back to the Teensy via the power line.
- The PM02 power module's 5V regulator provides isolation between battery noise and logic power -- battery VM connects only to the motor zones, never to the logic zone.
- **Motor PWM vs. IMU noise:** The 20 kHz PWM switching stays within the motor zones. The keep-out areas and star-ground topology ensure motor current return paths never flow under the IMU. If IMU readings still show PWM-frequency noise during testing:
  - Increase the keep-out gap width (3-5 mm).
  - Add ferrite beads on the GND star-ground bridges.
  - Verify no signal traces are routed across the keep-out gap without a ground guard.

### 5.6 Self-Balancing Control Loop Timing Considerations

The PCB design directly affects achievable control loop rates:

| Parameter | Value | Impact |
|-----------|-------|--------|
| SPI IMU read (14 bytes @ 7 MHz) | ~16 us | Fast; not the bottleneck |
| Teensy 4.1 PID computation | ~5-10 us | Negligible at 600 MHz |
| PWM update | < 1 us | Negligible |
| Encoder read (hardware counter) | < 1 us | Negligible |
| ADC read (2x IPROPI) | ~10 us | Optional per loop |
| **Total loop time** | **~30 us** | **Supports up to 30+ kHz loop rate** |

- The SPI upgrade from I2C reduces IMU read time from ~350 us (14 bytes @ 400 kHz I2C with overhead) to ~16 us, critical for high-rate balance control.
- Target control loop rate: **500-1000 Hz** for a responsive balancer. The hardware easily supports this.
- Use the IMU's data-ready interrupt (Pin 17) to trigger reads at the sensor's output data rate (up to 1.125 kHz for accel/gyro on ICM-20948) rather than polling.

---

## 6. Firmware Initialization Reference

```cpp
#include <SPI.h>

// === Pin Definitions ===

// IMU (SPI0)
#define IMU_CS      10    // ICM-20948 SPI chip select (SPI0 CS0)
#define IMU_INT     17    // ICM-20948 data-ready interrupt
// SPI0 hardware pins: SCK=13, MOSI=11, MISO=12

// PM02 Power Sensing
#define V_SENSE_PIN A0    // Battery voltage sense (pin 14)
#define I_SENSE_PIN A1    // Battery current sense (pin 15)

// Motor A (left motor zone, Pololu DRV8874 carrier A)
#define PWM_A       2     // Pololu A: EN/IN1 - speed (FlexPWM4_PWMA02)
#define DIR_A       3     // Pololu A: PH/IN2 - direction
#define SLEEP_A     29    // Pololu A: SLEEP - enable (active HIGH)
#define ENC_A_CHA   0     // Encoder A, channel A
#define ENC_A_CHB   1     // Encoder A, channel B
#define CS_A        A6    // Pololu A: CS - current sense ~1.1V/A (pin 20)
#define FAULT_A     33    // Pololu A: FAULT - active low, ext pullup

// Motor B (right motor zone, Pololu DRV8874 carrier B)
#define PWM_B       5     // Pololu B: EN/IN1 - speed (FlexPWM2_PWMA01)
#define DIR_B       6     // Pololu B: PH/IN2 - direction
#define SLEEP_B     30    // Pololu B: SLEEP - enable (active HIGH)
#define ENC_B_CHA   31    // Encoder B, channel A
#define ENC_B_CHB   32    // Encoder B, channel B
#define CS_B        A7    // Pololu B: CS - current sense ~1.1V/A (pin 21)
#define FAULT_B     34    // Pololu B: FAULT - active low, ext pullup

// ICM-20948 SPI settings: Mode 0 (CPOL=0, CPHA=0), up to 7 MHz
SPISettings imuSPISettings(7000000, MSBFIRST, SPI_MODE0);

void setup() {
    // --- Motor control (Pololu DRV8874 carriers) ---
    pinMode(SLEEP_A, OUTPUT);
    pinMode(SLEEP_B, OUTPUT);
    digitalWrite(SLEEP_A, LOW);   // Start both in sleep mode
    digitalWrite(SLEEP_B, LOW);   // PMODE latches on SLEEP rising edge
    pinMode(DIR_A, OUTPUT);
    pinMode(DIR_B, OUTPUT);
    analogWriteFrequency(PWM_A, 20000);  // 20 kHz PWM
    analogWriteFrequency(PWM_B, 20000);
    analogWriteResolution(10);            // 0-1023

    // Fault inputs (active low, external pullup R4 to 3.3V)
    pinMode(FAULT_A, INPUT);
    pinMode(FAULT_B, INPUT);

    // --- IMU (SPI @ 7MHz) ---
    pinMode(IMU_CS, OUTPUT);
    digitalWrite(IMU_CS, HIGH);  // Deselect IMU
    SPI.begin();                 // Initializes SCK=13, MOSI=11, MISO=12
    pinMode(IMU_INT, INPUT_PULLUP);

    // --- ADC ---
    analogReadResolution(12);  // Teensy 4.1 supports 12-bit ADC

    // --- Encoders ---
    pinMode(ENC_A_CHA, INPUT);
    pinMode(ENC_A_CHB, INPUT);
    pinMode(ENC_B_CHA, INPUT);
    pinMode(ENC_B_CHB, INPUT);

    // --- Wake up motor drivers ---
    // PMODE is tied LOW on PCB (PH/EN mode), latched on SLEEP rising edge
    delay(100);  // Let system stabilize
    digitalWrite(SLEEP_A, HIGH);  // Wake driver A (PMODE latches here)
    digitalWrite(SLEEP_B, HIGH);  // Wake driver B
    delay(2);    // Wait for tWAKE (1ms typ)
}

// Read motor current in amps from Pololu CS pin
// Pololu carrier has 2.49k RIPROPI -> CS output is ~1.1 V/A
float readMotorCurrent(int csPin) {
    int raw = analogRead(csPin);
    float voltage = raw * (3.3 / 4095.0);             // 12-bit ADC
    float current = voltage / (2490.0 * 455e-6);       // V / (RIPROPI * AIPROPI)
    return current;                                     // ~= voltage / 1.133
}

// Check fault status (returns true if fault detected)
bool checkFault(int faultPin) {
    return digitalRead(faultPin) == LOW;  // FAULT is open-drain, active low
}

// Example: Read a single register from ICM-20948 via SPI
uint8_t imuReadRegister(uint8_t reg) {
    uint8_t val;
    SPI.beginTransaction(imuSPISettings);
    digitalWrite(IMU_CS, LOW);
    SPI.transfer(reg | 0x80);  // Bit 7 = 1 for read
    val = SPI.transfer(0x00);  // Clock out data
    digitalWrite(IMU_CS, HIGH);
    SPI.endTransaction();
    return val;
}

// Example: Write a single register to ICM-20948 via SPI
void imuWriteRegister(uint8_t reg, uint8_t val) {
    SPI.beginTransaction(imuSPISettings);
    digitalWrite(IMU_CS, LOW);
    SPI.transfer(reg & 0x7F);  // Bit 7 = 0 for write
    SPI.transfer(val);
    digitalWrite(IMU_CS, HIGH);
    SPI.endTransaction();
}
```

---

## 7. Schematic Checklist

### Electrical -- Logic Zone
- [ ] Teensy 4.1 VIN connected to PM02 5V (J1 Pin 1) with decoupling (100nF + 10uF)
- [ ] ICM-20948 VIN connected to Teensy 3.3V
- [ ] SPI0 bus connected: SCK=Pin13->SCL, MOSI=Pin11->SDA, MISO=Pin12->SDO, CS=Pin10->CS
- [ ] ICM-20948 CS pin actively driven (not floating) to select SPI mode
- [ ] IMU INT connected to Teensy Pin 17
- [ ] PM02 V_SENSE routed to Teensy A0 (Pin 14) with R1/R2 divider and filtering
- [ ] PM02 I_SENSE routed to Teensy A1 (Pin 15) with filtering
- [ ] All GND pins connected to logic zone ground pour

### Electrical -- Motor Zone A (Left, Pololu Carrier A)
- [ ] Pololu carrier A footprint correct: 13 through-hole pins, 0.1" pitch, 0.6" row spacing
- [ ] Pololu VIN connected to battery pour from J2
- [ ] Pololu GND connected to motor zone A ground pour
- [ ] C4a (100uF bulk) placed within 10mm of Pololu VIN pin
- [ ] Pololu PMODE tied to GND (PH/EN mode -- CRITICAL, do not leave floating)
- [ ] Pololu IMODE left unconnected (20k pulldown on carrier)
- [ ] Pololu VREF left unconnected (10k pullup to SLEEP on carrier)
- [ ] Pololu EN/IN1 <- Teensy Pin 2 (PWM_A)
- [ ] Pololu PH/IN2 <- Teensy Pin 3 (DIR_A)
- [ ] Pololu SLEEP <- Teensy Pin 29 (active HIGH enable)
- [ ] Pololu CS: R6a/C9a filter to Teensy A6 (Pin 20)
- [ ] Pololu FAULT: R4a (10k) pullup to 3.3V, routed to Teensy Pin 33
- [ ] Pololu OUT1 -> J3 Pin 6 (M1) with C7a (100nF) to GND
- [ ] Pololu OUT2 -> J3 Pin 1 (M2) with C8a (10nF) to GND
- [ ] Pololu VM left unconnected (not needed; power enters via VIN)
- [ ] Encoder: J3 Pin 2 (V) <- 3.3V; J3 Pin 3 (A) -> Teensy Pin 0; J3 Pin 4 (B) -> Teensy Pin 1

### Electrical -- Motor Zone B (Right, Pololu Carrier B)
- [ ] (Mirror of Motor Zone A with these pin substitutions:)
- [ ] Pololu EN/IN1 <- Teensy Pin 5 (PWM_B); PH/IN2 <- Teensy Pin 6 (DIR_B)
- [ ] Pololu SLEEP <- Teensy Pin 30; CS filter -> Teensy A7 (Pin 21); FAULT -> Teensy Pin 34
- [ ] Motor outputs -> J4 (Pin 6 = M1, Pin 1 = M2) with EMI caps
- [ ] Encoder: J4 Pin 3 (A) -> Teensy Pin 31; J4 Pin 4 (B) -> Teensy Pin 32
- [ ] All other Pololu carrier wiring identical to Zone A

### Layout -- Zoned Ground Plane
- [ ] Three separate ground pours: Logic (center), Motor A (left), Motor B (right)
- [ ] Keep-out areas (2-3 mm gap) between logic pour and each motor pour
- [ ] Single star-ground point connecting all three pours, near J1/J2
- [ ] Zone-crossing traces grouped tightly through keep-out gaps
- [ ] IPROPI traces ground-guarded through keep-out areas
- [ ] GND bridge trace (1.0 mm) at each keep-out crossing point
- [ ] No traces routed under IMU footprint on ground plane layer
- [ ] VM copper pour from J2 to both motor zones (1.5 mm min or pour)

### Layout -- Self-Balancing
- [ ] IMU placed at board center, aligned above wheel axle when mounted
- [ ] IMU X-axis aligned with robot forward/backward (pitch) direction
- [ ] Solid unbroken logic ground pour under IMU footprint
- [ ] IMU offset from wheel axle center < 30 mm
- [ ] Board is left-right symmetric (Motor A zone mirrors Motor B zone)
- [ ] J3 on left edge, J4 on right edge (symmetric motor cable routing)
- [ ] 4x M3 mounting holes, symmetric, with space for rubber grommets
- [ ] Board dimensions as compact as possible (minimize moment of inertia)

### Mechanical / Chassis Integration
- [ ] Rubber grommets or soft-mount standoffs between PCB and chassis
- [ ] Battery mounted above wheel axle (high CoG for slower pendulum dynamics)
- [ ] All cables strain-relieved to chassis (not to PCB)
- [ ] Motor cables routed symmetrically left-to-right
- [ ] Hiwonder PH2.0 motor cables plug directly into J3/J4

---

## Appendix A: Adafruit ICM-20948 Breakout (4554) -- KiCad Footprint & Symbol Reference

Source: Official Adafruit EagleCAD board files (Revision C) from `github.com/adafruit/Adafruit-ICM20948-PCB`.

### A.1 Board Dimensions

| Parameter | Value (mm) | Value (inches) |
|-----------|-----------|----------------|
| Board width | 25.40 | 1.000 |
| Board height | 17.78 | 0.700 |
| Corner radius | 2.54 | 0.100 |
| Board origin | Bottom-left corner | |

### A.2 Through-Hole Pin Headers

| Parameter | Value |
|-----------|-------|
| Pitch | 2.54 mm (0.100") |
| Drill diameter | 1.0 mm |
| Pad diameter | 1.778 mm |
| Pins per row | 6 |
| Number of rows | 2 (top and bottom) |
| Row-to-row spacing (Y, center-to-center) | 12.70 mm (0.500") |
| Pin span within row (X, pin 1 to pin 6) | 12.70 mm (0.500") |
| First pin X (from left edge) | 6.35 mm (0.250") |
| Last pin X (from left edge) | 19.05 mm (0.750") |

### A.3 Pin Positions and Signals

**Bottom Row (JP1) -- Y = 2.54 mm (0.100" from bottom edge):**

Pin numbering left-to-right:

| Pin | Signal | KiCad Electrical Type | Direction | X (mm) | Y (mm) | Notes |
|-----|--------|-----------------------|-----------|--------|--------|-------|
| 1 | VIN | Power Input | In | 6.35 | 2.54 | 3-5V supply to on-board regulator |
| 2 | 1V8 | Power Output | Out | 8.89 | 2.54 | 1.8V regulated output, 100 mA max |
| 3 | GND | Power Input | In | 11.43 | 2.54 | Ground |
| 4 | SCK / SCL | Input | In | 13.97 | 2.54 | SPI clock / I2C clock |
| 5 | SDI / SDA | Bidirectional | In/Out | 16.51 | 2.54 | SPI MOSI (in) / I2C data (bidir) |
| 6 | INT | Output (open-drain) | Out | 19.05 | 2.54 | Interrupt, active low, needs pullup |

**Top Row (JP3) -- Y = 15.24 mm (0.100" from top edge):**

Pin numbering is **MIRRORED** (pin 1 on the RIGHT side):

| Pin | Signal | KiCad Electrical Type | Direction | X (mm) | Y (mm) | Notes |
|-----|--------|-----------------------|-----------|--------|--------|-------|
| 1 | CS | Input | In | 19.05 | 15.24 | SPI chip select, active low |
| 2 | SDO / ADR | Tri-state | Out | 16.51 | 15.24 | SPI MISO (out) / I2C addr select |
| 3 | GND | Power Input | In | 13.97 | 15.24 | Ground (second pin) |
| 4 | AUX_SCL (AC) | Bidirectional | In/Out | 11.43 | 15.24 | Aux I2C clock, **1.8V logic only** |
| 5 | AUX_SDA (AD) | Bidirectional | In/Out | 8.89 | 15.24 | Aux I2C data, **1.8V logic only** |
| 6 | FSYNC (FS) | Input | In | 6.35 | 15.24 | Frame sync, **1.8V logic only**. Tie to GND if unused |

> **WARNING:** The top row pin numbering is mirrored compared to the bottom row.
> Pin 1 (CS) is on the RIGHT, pin 6 (FSYNC) is on the LEFT. This is the most
> common mistake when creating the footprint. Double-check by looking at the
> Adafruit board silkscreen -- CS is at the top-right corner.

### A.4 Pin Diagram (Top View)

```
                    25.40 mm (1.000")
    <--------------------------------------------------->

    +---------------------------------------------------+  ---
    | (MH)  FS   AD   AC   GND  SDO  CS          (MH)  |   ^
    | 2.54  6.35 8.89 11.43 13.97 16.51 19.05   22.86  |   |
    | 15.24  <--- Top Row (JP3), pin 1 at RIGHT --->    |   |
    |                                                    |   |
    | [STEMMA QT]     [ ICM-20948 ]       [STEMMA QT]   |  17.78 mm
    |  left, x=2.54    center of           right,x=22.86|  (0.700")
    |                    board                           |   |
    |                                                    |   |
    | 2.54  <--- Bottom Row (JP1), pin 1 at LEFT --->   |   |
    | (MH)  VIN  1V8  GND  SCL  SDA  INT         (MH)  |   |
    | 2.54  6.35 8.89 11.43 13.97 16.51 19.05   22.86  |   v
    +---------------------------------------------------+  ---

    (MH) = Mounting Hole (2.5mm drill, 3.2mm pad)

    Coordinate reference:
      Origin (0, 0) = bottom-left corner
      All X/Y values in mm from origin
```

### A.5 Mounting Holes

| Location | X (mm) | Y (mm) | Drill (mm) | Pad (mm) |
|----------|--------|--------|------------|----------|
| Bottom-left | 2.54 | 2.54 | 2.5 | 3.2 |
| Bottom-right | 22.86 | 2.54 | 2.5 | 3.2 |
| Top-left | 2.54 | 15.24 | 2.5 | 3.2 |
| Top-right | 22.86 | 15.24 | 2.5 | 3.2 |

Note: Mounting holes share the same Y coordinates as the pin header rows. They are inset 2.54 mm from the left/right board edges.

### A.6 STEMMA QT / Qwiic Connectors (SMD, JST SH 4-pin)

These are 1.0 mm pitch SMD connectors on the left and right board edges. For the main PCB footprint, you can represent these as part of the courtyard/silkscreen outline but they don't need through-hole pads on your carrier PCB (the Adafruit breakout is a module you solder onto your board via the through-hole headers).

| Connector | X center (mm) | Y center (mm) | Orientation |
|-----------|---------------|---------------|-------------|
| Left | 2.54 | 8.89 | Vertical (270 deg) |
| Right | 22.86 | 8.89 | Vertical (90 deg) |

Signal order: GND, VIN, SDA, SCL (both connectors, same I2C bus as the pin headers).

### A.7 KiCad Footprint Checklist

- [ ] Board outline: 25.40 x 17.78 mm rectangle on Fab layer, 2.54 mm corner radius
- [ ] Bottom row: 6 pads, pitch 2.54 mm, starting at (6.35, 2.54), drill 1.0 mm, pad 1.778 mm
- [ ] Top row: 6 pads, pitch 2.54 mm, starting at (6.35, 15.24), drill 1.0 mm, pad 1.778 mm
- [ ] Top row pin 1 (CS) is at X=19.05 (RIGHT side), pin 6 (FSYNC) is at X=6.35 (LEFT side)
- [ ] 4 mounting holes: drill 2.5 mm, pad 3.2 mm, at corners (see table above)
- [ ] Pin 1 markers on silkscreen for both rows
- [ ] Courtyard slightly larger than board outline (e.g., 0.5 mm clearance)
- [ ] Reference designator and value on Fab layer

### A.8 KiCad Symbol Checklist

**Required pins (active in our SPI design):**

| Pin Name | Symbol Side | Electrical Type | Connected To |
|----------|------------|-----------------|-------------|
| VIN | Left | Power input | Teensy 3.3V |
| GND | Left | Power input | GND |
| SCK/SCL | Right | Input | Teensy Pin 13 (SCK0) |
| SDI/SDA | Right | Input | Teensy Pin 11 (MOSI0) |
| SDO | Right | Output | Teensy Pin 12 (MISO0) |
| CS | Right | Input | Teensy Pin 10 (CS0) |
| INT | Right | Output (open-drain) | Teensy Pin 17 |

**Optional / unused pins (include in symbol but may be left unconnected):**

| Pin Name | Symbol Side | Electrical Type | Notes |
|----------|------------|-----------------|-------|
| 1V8 | Left | Power output | 1.8V regulator output, 100 mA max |
| AUX_SCL (AC) | Right | Bidirectional | Auxiliary I2C, 1.8V logic only |
| AUX_SDA (AD) | Right | Bidirectional | Auxiliary I2C, 1.8V logic only |
| FSYNC (FS) | Right | Input | Frame sync, 1.8V logic only. Tie to GND if unused. |
| GND (top row) | Left | Power input | Second GND pin on top row |

---

## Appendix B: Pololu DRV8874 Carrier (#4035) -- KiCad Footprint & Symbol Reference

Source: Official Pololu dimension drawing (PDF), DXF drill guide (dev code md41a), and product page photos.

### B.1 Board Dimensions

| Parameter | Value (mm) | Value (inches) |
|-----------|-----------|----------------|
| Board width | 15.20 | 0.600 |
| Board height | 17.80 | 0.700 |
| Board thickness | 1.80 | 0.071 |
| Corner style | Sharp corners (top-right may be slightly chamfered) | |

### B.2 Through-Hole Pin Headers

| Parameter | Value |
|-----------|-------|
| Total pins | **14** (7 per row) |
| Pitch | 2.54 mm (0.100") |
| Drill diameter | 1.02 mm (0.040") |
| Suggested pad diameter | 1.778 mm (standard for 1.02mm drill) |
| Number of rows | 2 (left and right) |
| Row-to-row spacing (X, center-to-center) | **12.70 mm (0.500")** |
| Row inset from board edge | 1.27 mm (0.050") each side |
| Pin span within row (Y, pin 1 to pin 7) | 15.24 mm (0.600") |
| First pin Y (from bottom edge) | 1.27 mm (0.050") |
| Last pin Y (from bottom edge) | 16.51 mm (0.650") |
| Mounting holes | **None** |

### B.3 Pin Positions and Signals

**Left Row -- Logic/Control Side (X = 1.27 mm from left edge):**

Pin numbering top-to-bottom:

| Pin | Signal | KiCad Elec. Type | Direction | X (mm) | Y (mm) | Notes |
|-----|--------|-------------------|-----------|--------|--------|-------|
| 1 | VM | Power Output | Out | 1.27 | 16.51 | Motor supply after reverse-voltage MOSFET |
| 2 | GND | Power Input | In | 1.27 | 13.97 | Ground |
| 3 | EN/IN1 | Input | In | 1.27 | 11.43 | PWM speed control (PH/EN mode) |
| 4 | PH/IN2 | Input | In | 1.27 | 8.89 | Direction control (PH/EN mode) |
| 5 | PMODE | Input | In | 1.27 | 6.35 | Mode select. **Tie LOW for PH/EN mode** |
| 6 | SLEEP | Input | In | 1.27 | 3.81 | Enable, active HIGH. Internal 100k pulldown |
| 7 | VREF | Input | In | 1.27 | 1.27 | Current limit ref. 10k pullup to SLEEP on-board |

**Right Row -- Motor/Power Side (X = 13.97 mm from left edge):**

Pin numbering top-to-bottom:

| Pin | Signal | KiCad Elec. Type | Direction | X (mm) | Y (mm) | Notes |
|-----|--------|-------------------|-----------|--------|--------|-------|
| 1 | VIN | Power Input | In | 13.97 | 16.51 | 4.5-37V motor supply input (reverse protected) |
| 2 | GND | Power Input | In | 13.97 | 13.97 | Ground (second GND pin) |
| 3 | OUT1 | Output | Out | 13.97 | 11.43 | Motor output 1 |
| 4 | OUT2 | Output | Out | 13.97 | 8.89 | Motor output 2 |
| 5 | IMODE | Input | In | 13.97 | 6.35 | OCP mode. 20k pulldown on-board |
| 6 | FAULT | Open Collector | Out | 13.97 | 3.81 | Fault flag, active low. **No pullup on-board** |
| 7 | CS | Output | Out | 13.97 | 1.27 | Current sense, ~1.1 V/A. 2.49k to GND on-board |

> **The "14th pin" mystery:** The Pololu product page lists 13 named pins but the board
> has 14 holes. The 14th is a **second GND** -- one on each side (left pin 2, right pin 2).
> Both are the same net.

### B.4 Pin Diagram (Top View)

```
                15.20 mm (0.600")
    <----------------------------------->

    +-----------------------------------+  ---
    |                                   |   ^
    | VM  (1L)                (1R) VIN  |   |
    | 1.27                       13.97  |   |
    | 16.51                      16.51  |   |
    |                                   |   |
    | GND (2L)                (2R) GND  |   |
    | 13.97                      13.97  |   |
    |                                   |   |
    | EN  (3L)      [DRV8874]  (3R) OUT1|   |
    | 11.43            IC        11.43  |  17.80 mm
    |                                   |  (0.700")
    | PH  (4L)                (4R) OUT2 |   |
    | 8.89                        8.89  |   |
    |                                   |   |
    | PMODE(5L)             (5R) IMODE  |   |
    | 6.35                        6.35  |   |
    |                                   |   |
    | SLP (6L)              (6R) FAULT  |   |
    | 3.81                        3.81  |   |
    |                                   |   |
    | VREF(7L)                (7R) CS   |   |
    | 1.27                        1.27  |   v
    +-----------------------------------+  ---

    X coords:  1.27 (left)    13.97 (right)
    Y coords shown next to each pin
    Row-to-row: 12.70 mm (0.500")
    All dimensions in mm, origin at bottom-left
```

### B.5 Suggested Pin Numbering for KiCad (Symbol <-> Footprint)

Use sequential numbering: left row 1-7, then right row 8-14.

| Pad # | Row | Signal | Position (X, Y) mm |
|-------|-----|--------|---------------------|
| 1 | Left | VM | (1.27, 16.51) |
| 2 | Left | GND | (1.27, 13.97) |
| 3 | Left | EN/IN1 | (1.27, 11.43) |
| 4 | Left | PH/IN2 | (1.27, 8.89) |
| 5 | Left | PMODE | (1.27, 6.35) |
| 6 | Left | SLEEP | (1.27, 3.81) |
| 7 | Left | VREF | (1.27, 1.27) |
| 8 | Right | VIN | (13.97, 16.51) |
| 9 | Right | GND | (13.97, 13.97) |
| 10 | Right | OUT1 | (13.97, 11.43) |
| 11 | Right | OUT2 | (13.97, 8.89) |
| 12 | Right | IMODE | (13.97, 6.35) |
| 13 | Right | FAULT | (13.97, 3.81) |
| 14 | Right | CS | (13.97, 1.27) |

### B.6 KiCad Footprint Checklist

- [ ] Board outline: 15.20 x 17.80 mm rectangle on Fab layer
- [ ] Left row: 7 pads at X=1.27, Y from 16.51 to 1.27, pitch 2.54 mm, drill 1.02 mm
- [ ] Right row: 7 pads at X=13.97, Y from 16.51 to 1.27, pitch 2.54 mm, drill 1.02 mm
- [ ] Row-to-row spacing: 12.70 mm (X: 13.97 - 1.27)
- [ ] Pad numbering: 1-7 left (top to bottom), 8-14 right (top to bottom)
- [ ] Pin 1 marker on silkscreen (VM, top-left)
- [ ] Courtyard 0.25-0.5 mm larger than board outline
- [ ] No mounting holes (board has none)

### B.7 KiCad Symbol Pin Table

**Left side of symbol (Logic/Control pins):**

| Pin # | Pin Name | Electrical Type | Notes |
|-------|----------|-----------------|-------|
| 1 | VM | Power Output | Motor supply after reverse-voltage protection |
| 2 | GND | Power Input | Ground |
| 3 | EN/IN1 | Input | PWM speed (PH/EN mode) |
| 4 | PH/IN2 | Input | Direction (PH/EN mode) |
| 5 | PMODE | Input | Mode select. Tie LOW for PH/EN |
| 6 | SLEEP | Input | Enable, active HIGH |
| 7 | VREF | Input | Current limit ref (10k to SLEEP on-board) |

**Right side of symbol (Motor/Power pins):**

| Pin # | Pin Name | Electrical Type | Notes |
|-------|----------|-----------------|-------|
| 8 | VIN | Power Input | 4.5-37V motor supply |
| 9 | GND | Power Input | Ground (2nd GND) |
| 10 | OUT1 | Output | Motor terminal 1 |
| 11 | OUT2 | Output | Motor terminal 2 |
| 12 | IMODE | Input | OCP mode (20k pulldown on-board) |
| 13 | FAULT | Open Collector | Fault flag, active low. **Add ext. pullup** |
| 14 | CS | Output | Current sense ~1.1 V/A (2.49k on-board) |

### B.8 What's On-Board vs External (Quick Reference)

| On Pololu Carrier (do NOT add) | External (you MUST add) |
|-------------------------------|------------------------|
| DRV8874 IC (pre-soldered) | PMODE tie to GND (trace on main PCB) |
| VM bypass cap (100nF) | FAULT pullup R (10k to 3.3V) |
| Charge pump caps (storage + flying) | VM bulk cap (100uF, near VIN) |
| Reverse-voltage MOSFET | Motor EMI caps (100nF + 10nF at JST-PH) |
| IMODE pulldown (20k) | CS low-pass filter (R+C, optional) |
| VREF pullup (10k to SLEEP) | |
| CS/IPROPI resistor (2.49k) | |
