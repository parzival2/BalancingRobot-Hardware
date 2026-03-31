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
|  +-------------+  :                           :  +-------------+          |
|  [nFAULT R]       :                           :  [nFAULT R]              |
|  [IPROPI filter]  +---------keep-out---------+   [IPROPI filter]         |
|  [Motor EMI caps]                                [Motor EMI caps]         |
|                                                                           |
|  [J3 Motor+Enc A]       [J1 PM02D Power]         [J4 Motor+Enc B]       |
|                           [J2 Battery VM]                                 |
|   (M3)               (M3)            (M3)               (M3)             |
+===========================================================================+

  All connections between zones are PCB traces (no cables needed).
  Keep-out areas: no copper pour, no traces (acts as noise barrier).
  Each zone has its own copper pour with GND bridges at keep-out crossings.
  Pololu carriers soldered directly via 0.1" pin headers for vibration resistance.
```

**External connectors:**

| Schematic Ref | Type | Function | Current | Location |
|---------------|------|----------|---------|----------|
| **J1** | JST-PH 6-pin (S6B-PH-K-S) | PM02D power (5.2V) + I2C sense | ~0.3A | Bottom center |
| **J3** | XT30 male PCB-mount | Battery VM input (7.4V raw) | **2.8A peak** | Bottom center |
| **J4** | JST-PH 6-pin (S6B-PH-K-S) | Motor A + Encoder A (Hiwonder) | 1.4A motor + signal | Left edge |
| **J5** | JST-PH 6-pin (S6B-PH-K-S) | Motor B + Encoder B (Hiwonder) | 1.4A motor + signal | Right edge |
| **J2** | JST-PH 4-pin (S4B-PH-K-S) | UART2 (companion board: RPi/BBB) | Signal only | Top edge |

> **Why XT30 for J2?** The battery VM input carries up to 2.8A peak (both motors at stall).
> JST-PH is only rated for 2A per contact -- insufficient for motor power. XT30 is rated
> for 30A, is compact (similar footprint to a 2-pin JST-XH), and is widely available at
> any RC hobby shop.
>
> **Why JST-PH for J1?** The PM02D comes with a Molex CLIK-Mate cable, but Molex headers
> are hard to source. Cut the PM02D cable and re-crimp JST-PH contacts onto the 6 wires.
> The current on J1 is only ~0.3A (Teensy + accessories), well within JST-PH's 2A rating.
>
> **J3/J4 motor current through JST-PH:** Each motor draws 1.4A at stall. The motor power
> pins (M1, M2) on J3/J4 are within the JST-PH 2A rating for normal operation (0.65A rated).
> At stall, 1.4A is under the 2A limit. If you want more margin, the Hiwonder motors come
> with PH2.0 cables which are JST-PH compatible -- so this is matched to the motor's own
> connector rating.

---

## 1. Component & Connector List (BOM)

### Single Unified PCB

**Logic Zone (Center)**

| Ref | Component | Description | Qty |
|-----|-----------|-------------|-----|
| U1 | Teensy 4.1 | ARM Cortex-M7 @ 600 MHz, with pin headers | 1 |
| U2 | Adafruit ICM-20948 | 9-DoF IMU breakout (Accel/Gyro/Mag) | 1 |
| J1 | JST-PH 6-pin (S6B-PH-K-S) | PM02D power + I2C (re-crimp PM02D cable) | 1 |
| J2 | JST-PH 4-pin (S4B-PH-K-S) | UART2 header for companion board (RPi/BBB) | 1 |
| C1 | 100 nF ceramic | Decoupling on Teensy VIN | 1 |
| -- | **10 uF electrolytic (MISSING from schematic)** | **Bulk cap on Teensy VIN -- ADD THIS** | 1 |

**Motor Zone A (Left) -- Identical to Motor Zone B**

The Pololu DRV8874 carrier (15.2 x 17.8 mm) already includes: DRV8874 IC, VM bypass cap, charge pump caps (storage + flying), reverse-voltage protection MOSFET, IMODE 20k pulldown, VREF 10k pullup to SLEEP, and RIPROPI 2.49k to GND (CS output at ~1.1 V/A). The only external components needed per zone are:

| Schematic Ref (Zone A / Zone B) | Component | Description | Qty each |
|----------------------------------|-----------|-------------|----------|
| U3 / U4 | Pololu DRV8874 carrier (#4035) | Motor driver breakout, 0.1" headers | 1 |
| J4 / J5 | JST-PH 6-pin (S6B-PH-K-S) | Motor + encoder connector | 1 |
| -- / -- | **100 uF 16V electrolytic (MISSING)** | **VM bulk cap near Pololu VIN -- ADD THIS** | 1 |
| C3 / C5 | 100 nF ceramic | Motor terminal EMI cap (OUT1/M1) | 1 |
| C4 / C6 | 10 nF ceramic | Motor terminal EMI cap (OUT2/M2) | 1 |
| R1 / R3 | 10 kOhm | FAULT pullup to 3.3V (not on Pololu board) | 1 |
| R2 / R4 | 1 kOhm | CS low-pass filter resistor (for torque control) | 1 |
| C2 / -- | 10 nF ceramic | CS low-pass filter cap (fc = 15.9 kHz with 1k) | 1 |

> **NOTE: Schematic currently shows C2 as 10nF. This should be the CS filter cap for Motor A.
> A second CS filter cap for Motor B is needed but not yet placed in the schematic.**

**Shared / Board-Level**

| Ref | Component | Description | Qty |
|-----|-----------|-------------|-----|
| J3 | XT30 male PCB-mount | Battery VM input (7.4V, 2.8A peak, splits to both zones) | 1 |
| FB1, FB2 | Ferrite bead 600 Ohm @ 100 MHz | 3.3V isolation into each motor zone (optional) | 2 |
| -- | 0.1" pin headers (13-pin) | For mounting each Pololu carrier to main PCB | 2 sets |
| -- | M3 mounting holes (3.2 mm) | Board corners, symmetric | 4 |

> **What the Pololu carrier eliminates from the BOM (per zone):** VM bypass cap (100nF),
> charge pump storage cap (100nF), charge pump flying cap (22nF), RIPROPI sense resistor
> (2.49k on-board), IMODE pulldown resistor (20k on-board), VREF pullup resistor (10k
> on-board to SLEEP), and the bare DRV8874 HTSSOP-16 IC itself. That is 7 SMD components
> per zone (14 total) replaced by one drop-in module.

---

## 2. Teensy 4.1 Pin Layout Diagram

Complete pin-to-function mapping for this design. Left side faces the ICM-20948 IMU when the Teensy is rotated 90 degrees on the PCB.

```
Teensy 4.1 Pin Layout -- Current Design
(showing all pin assignments and SPI alternate options)

                 LEFT SIDE                          RIGHT SIDE
                 (faces IMU)                        (away from IMU)

            GND  ●                          ● VIN
       (0)  D0   ●  ← ENC_A_PH_A           ● GND
       (1)  D1   ●  ← ENC_A_PH_B [MISO1]   ● 3V3
       (2)  D2   ●  ← MOTA_PWM              ● D23
       (3)  D3   ●  ← MOTA_DIR              ● D22
       (4)  D4   ●  free                    ● D21 (A7) ← MOTB_CUR
       (5)  D5   ●  ← MOTB_PWM              ● D20 (A6) ← MOTA_CUR
       (6)  D6   ●  ← MOTB_DIR              ● D19 (SCL) ← PM02_SCL
       (7)  D7   ●  ← UART_RX               ● D18 (SDA) ← PM02_SDA
       (8)  D8   ●  ← UART_TX               ● D17 ← ICM20948_INT
       (9)  D9   ●  free                    ● D16
      (10)  D10  ●  ← SPI CS  ✓             ● D15
      (11)  D11  ●  ← SPI MOSI ✓            ● D14
      (12)  D12  ●  ← SPI MISO ✓            ● D13 ← SPI SCK ✗ (wrong side!)
            3V3  ●                          ● A14(38)
      (24)  D24  ●  free                    ● A15(39)
      (25)  D25  ●  free                    ● A16(40)
      (26)  D26  ●  free [MOSI1]            ● A17(41)
      (27)  D27  ●  free [SCK1]             ● D33 ← MOTA_FAULT
      (28)  D28  ●  free                    ● D34 ← MOTB_FAULT
      (29)  D29  ●  ← MOTA_SLEEP            ● D35
      (30)  D30  ●  ← MOTB_SLEEP            ● D36
      (31)  D31  ●  ← ENC_B_PH_A            ● D37
      (32)  D32  ●  ← ENC_B_PH_B            ●

  ✓ = SPI pin on correct side (short trace to IMU)
  ✗ = SPI pin on wrong side (long trace, 37mm routed)
  [MOSI1] [SCK1] [MISO1] = SPI1 alternate pin options (see Section 3.2)
```

---

## 2b. Complete Pin Mapping Table

### Teensy 4.1 Master Pin Assignment

| Teensy Pin | Arduino Name | Function | Connected To | Notes |
|------------|-------------|----------|--------------|-------|
| **VIN** | VIN | 5V power input | J1 Pins 1+2 (PM02D VCC) | 3.6-5.5V; PM02D outputs 5.2V |
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
| | | **--- PM02D Power (I2C) ---** | | |
| **19** | A5 / SCL0 | I2C0 SCL | J1 Pin 3 (PM02D SCL) | INA226 voltage/current monitor |
| **18** | A4 / SDA0 | I2C0 SDA | J1 Pin 4 (PM02D SDA) | INA226 I2C address 0x40 |
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
| | | | | |
| | | **--- UART2 (Companion Board) ---** | | |
| **8** | D8 / TX2 | UART TX | J5 Pin 1 (TX) | Serial2 TX, up to 6 Mbit/s |
| **7** | D7 / RX2 | UART RX | J5 Pin 2 (RX) | Serial2 RX, up to 6 Mbit/s |

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

### PM02D Power Module (J1) -- Molex CLIK-Mate 6-Pin

The Holybro PM02D uses a **Molex CLIK-Mate 2.0mm pitch** connector (not JST-PH). The PM02D provides regulated 5.2V power and battery voltage/current monitoring via **I2C** (using an on-board INA226 power monitor IC).

**Connector:** JST-PH 6-pin, S6B-PH-K-S (2.0mm pitch, vertical, through-hole)
**Cable:** Cut the PM02D's stock Molex CLIK-Mate connector off the cable and re-crimp with JST-PH contacts (SPH-002T-P0.5S) into a PHR-6 housing. Maintain the original wire order.

| J1 Pin | Signal | Wire Color (typical) | Connected To | Notes |
|--------|--------|---------------------|-------------|-------|
| 1 | VCC | Red | Teensy VIN | 5.2V regulated, 3A max |
| 2 | VCC | Red | Teensy VIN (parallel) | Both VCC pins tied together on PCB |
| 3 | SCL | Yellow | Teensy Pin 19 (SCL0) | I2C clock, 400 kHz |
| 4 | SDA | Green | Teensy Pin 18 (SDA0) | I2C data |
| 5 | GND | Black | Logic zone GND | |
| 6 | GND | Black | Logic zone GND (parallel) | Both GND pins tied together on PCB |

**Wiring diagram:**

```
PM02D Module                     J1 (Molex 6-pin)              Teensy 4.1
                                 on your PCB
  XT60 ←→ Battery
    |                           ┌──────────┐
    |    ┌──5.2V reg──→ Pin 1 (VCC) ──┬──────────────→ VIN
    |    |              Pin 2 (VCC) ──┘     C1(100nF)──┐
    |    |                              C2(10uF)──┐    │
    |    │              Pin 3 (SCL) ──────────────→ Pin 19 (SCL0 / Wire)
    └──INA226──→        Pin 4 (SDA) ──────────────→ Pin 18 (SDA0 / Wire)
         |              Pin 5 (GND) ──┬──────────────→ GND
         └──────→       Pin 6 (GND) ──┘               │
                        └──────────┘               GND ┘
```

**I2C details:**
- **IC:** INA226 power monitor (on PM02D board)
- **I2C address:** 0x40 (default)
- **I2C bus:** Wire / I2C0 (Teensy Pins 18/19). This bus is free because the IMU uses SPI.
- **Bus speed:** 400 kHz (Fast Mode)
- **Pull-ups:** The PM02D has on-board I2C pull-ups. No additional pull-ups needed on your PCB.
- **Data available:** Battery voltage (mV), battery current (mA), power (mW)
- **Polling rate:** 10-50 Hz is sufficient for battery monitoring (does not affect control loop)

> **Why I2C instead of analog?** The PM02D's INA226 provides higher accuracy (better than 5%)
> with no calibration needed -- no voltage divider ratios or mV/A sensitivities to measure.
> The PX4 INA226 driver reads voltage and current automatically and publishes to `battery_status`.

> **No interference with balance control:** The I2C bus (Pins 18/19) is completely independent
> of the SPI bus (Pins 10-13) used by the IMU. The INA226 driver runs on a low-priority work
> queue at 10-50 Hz. It cannot block or delay the 1 kHz balance loop or the 10-20 kHz current
> control loop.

### Battery Input (J2) -- XT30 Male PCB-Mount

XT30 is used here because the battery VM input carries up to 2.8A peak (both motors at stall). JST-PH is only rated for 2A per contact -- insufficient. XT30 is rated for 30A continuous.

| J2 Pin | Signal | Connected To | Notes |
|--------|--------|-------------|-------|
| + | VM+ | Motor zone A and B VM rails (via copper pour) | 7.4V raw battery, 2.8A peak |
| - | GND | Motor zone ground pours | |

**Cable:** Solder an XT30 female pigtail to the PM02D's XT60 battery pass-through output. Use 18-20 AWG silicone wire, keep under 150 mm.

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

### UART2 Companion Board Header (J5) -- JST-PH 4-Pin

For high-speed serial communication with a Raspberry Pi, BeagleBone, or other companion computer. Uses Teensy Serial2 (hardware UART, up to 6 Mbit/s).

| J5 Pin | Signal | Teensy Pin | Connected To | Notes |
|--------|--------|-----------|-------------|-------|
| 1 | TX | Pin 8 (TX2) | Companion RX | Teensy transmit -> companion receive |
| 2 | RX | Pin 7 (RX2) | Companion TX | Teensy receive <- companion transmit |
| 3 | GND | GND | Companion GND | Signal ground (required for UART) |
| 4 | 3V3 | 3.3V | Companion 3V3 ref | Optional: voltage reference, NOT for powering companion |

> **Voltage compatibility:** Teensy 4.1 UART is **3.3V logic**, which directly matches:
> - Raspberry Pi GPIO UART (3.3V) -- connect directly, no level shifter
> - BeagleBone GPIO UART (3.3V) -- connect directly, no level shifter
> - Arduino 5V boards -- **NEED level shifter** (Teensy pins are NOT 5V tolerant)
>
> **Pin 4 (3V3) is a reference only.** Do NOT use it to power the companion board
> (Teensy 3.3V output is limited to 250 mA). Power the RPi/BBB from its own supply.
> The 3V3 pin is provided so the companion can detect voltage level if needed.

**Wiring diagram:**
```
Teensy 4.1                J5 (JST-PH 4-pin)        Companion Board
                          on your PCB               (RPi / BBB)

Pin 8 (TX2) ─────────── Pin 1 (TX) ────────────── RX (GPIO input)
Pin 7 (RX2) ─────────── Pin 2 (RX) ────────────── TX (GPIO output)
GND ─────────────────── Pin 3 (GND) ───────────── GND
3V3 ─────────────────── Pin 4 (3V3) ───────────── (optional ref)
```

**Use cases:**
- **MAVLink telemetry** -- PX4 MAVLink over Serial2 to companion running MAVSDK/ROS2
- **Offboard control** -- Companion sends high-level commands (waypoints, velocity targets)
- **Sensor fusion** -- Companion sends camera/LIDAR data to Teensy for obstacle avoidance
- **Logging** -- High-speed data streaming to companion for real-time analysis

---

## 3. Netlist Wiring Guide -- Logic Zone (Center)

### 3.1 Power Distribution

```
PM02D 5.2V (J1 Pins 1+2) ----+---- C2 (10uF) ----+---- GND (logic zone)
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

### 3.4 PM02D Power Monitoring (I2C)

The PM02D uses I2C for battery voltage and current monitoring. No analog pins or divider resistors are needed.

```
J1 Pin 3 (SCL) ---- Teensy Pin 19 (SCL0 / Wire)
J1 Pin 4 (SDA) ---- Teensy Pin 18 (SDA0 / Wire)
```

- **I2C bus:** Wire / I2C0, 400 kHz (Fast Mode)
- **No external pull-ups needed** -- the PM02D has on-board I2C pull-ups.
- **I2C address:** 0x40 (INA226 default)
- **No bus conflict:** The IMU uses SPI (Pins 10-13), so I2C0 (Pins 18/19) is dedicated to the PM02D.
- Route SCL and SDA traces as a pair, away from PWM and motor traces.
- Keep traces within the logic zone -- they do not need to cross any keep-out area.

**In PX4, start the driver in `rc.board_sensors`:**
```bash
# PM02D power monitor on I2C bus 0, INA226 at address 0x40
ina226 -I -b 0 -a 64 start    # 64 = 0x40 decimal
```

This publishes to `battery_status` uORB topic. QGroundControl displays battery voltage, current, and percentage automatically.

**Freed pins:** Teensy A0 (Pin 14) and A1 (Pin 15) are now available for other uses (e.g., additional ADC inputs, or leave unconnected).

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

### 3.7 UART2 Companion Board Interface (J5)

```
Teensy Pin 8  (TX2) ---- J5 Pin 1 (TX)
Teensy Pin 7  (RX2) ---- J5 Pin 2 (RX)
GND                  ---- J5 Pin 3 (GND)
Teensy 3V3           ---- J5 Pin 4 (3V3)
```

- Serial2 (UART2): hardware UART, up to **6 Mbit/s**.
- Pins 7 and 8 have no conflicts with any other function in this design.
- Route TX/RX traces within the logic zone. Keep away from PWM and SPI traces.
- Place J5 on the **top edge** of the board for easy cable access to the companion board.
- In PX4, configure MAVLink on Serial2: `mavlink start -d /dev/ttyS1 -b 921600`

### 3.8 Traces Crossing Keep-Out Zones

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

## 4. DRV8874 Motor Driver Wiring -- Complete Diagram

Both motor zones are **mirror images**. The schematic is identical; only the Teensy pin numbers differ. This section shows the **complete wiring for one motor channel** with every pin accounted for.

### 4.1 What's On the Pololu Carrier vs What You Add

| On Pololu Carrier (do NOT add) | You Add on Main PCB |
|-------------------------------|---------------------|
| DRV8874 IC | C4: 100uF bulk cap near VIN |
| VM bypass cap (100nF) | R4: 10k FAULT pullup to 3.3V |
| Charge pump caps (storage + flying) | R6: 1k CS filter resistor |
| Reverse-voltage MOSFET | C9: 10nF CS filter cap |
| RIPROPI 2.49k to GND (CS = ~1.1 V/A) | C7: 100nF EMI cap at motor connector |
| IMODE 20k pulldown | C8: 10nF EMI cap at motor connector |
| VREF 10k pullup to SLEEP | PMODE trace to GND |

### 4.2 Complete Wiring Diagram (One Motor Channel)

```
                        MAIN PCB - ONE MOTOR ZONE
 ═══════════════════════════════════════════════════════════════════════════

   From J2 (XT30)                    POLOLU DRV8874 CARRIER
   Battery 7.4V                     ┌─────────────────────────┐
        │                           │                         │
        ├──── C4 (100uF) ── GND    │  LEFT ROW    RIGHT ROW  │
        │                           │  (Logic)     (Motor)    │
        │                           │                         │
        └───────────────────────────┤→ [1] VM      VIN [8] ←──┤── Battery 7.4V
                                    │                         │     (from J2)
   GND (motor zone pour) ──────────┤→ [2] GND     GND [9] ←──┤── GND
                                    │                         │
   Teensy Pin 2 (PWM_A) ───────────┤→ [3] EN/IN1  OUT1[10]───┤──┬── J3 Pin 6 (M1)
     or Pin 5 (PWM_B)              │                         │  │
                                    │                         │  C7 (100nF)
   Teensy Pin 3 (DIR_A) ───────────┤→ [4] PH/IN2  OUT2[11]───┤──┤── J3 Pin 1 (M2)
     or Pin 6 (DIR_B)              │                         │  │
                                    │                         │  C8 (10nF)
   GND (motor zone pour) ──────────┤→ [5] PMODE   IMODE[12]──┤── NC
                  ▲                 │        ▲                │  (20k pulldown
            CRITICAL!               │    Tie LOW!             │   on-board)
         Must be GND                │                         │
         NOT floating!              │                         │
                                    │                         │
   Teensy Pin 29 (SLEEP_A) ────────┤→ [6] SLEEP   FAULT[13]──┤──┬── Teensy Pin 33
     or Pin 30 (SLEEP_B)           │                         │  │   (FAULT_A)
                                    │                         │  │   or Pin 34
                                    │                         │  R4 (10k)
                                    │                         │  │
                                    │                    3.3V ┤──┘
                                    │                         │
                                    │  [7] VREF      CS [14]──┤──── R6 (1k) ──┬── Teensy Pin 20
                                    │       │                 │               │   (CS_A / A6)
                                    │      NC                 │          C9 (10nF) or Pin 21
                                    │  (10k pullup            │               │   (CS_B / A7)
                                    │   to SLEEP              │              GND
                                    │   on-board)             │
                                    └─────────────────────────┘


   MOTOR CONNECTOR (J3 or J4, JST-PH 6-pin, mates with Hiwonder cable)
   ┌──────────────────────────────────────────┐
   │  Pin 1: M2 (motor -) ←── OUT2 + C8      │
   │  Pin 2: V  (encoder 3.3V) ←── 3.3V      │
   │  Pin 3: A  (encoder phase A) ──→ Teensy  │  Pin 0 (Enc_A) or Pin 31 (Enc_B)
   │  Pin 4: B  (encoder phase B) ──→ Teensy  │  Pin 1 (Enc_A) or Pin 32 (Enc_B)
   │  Pin 5: G  (encoder GND) ←── GND         │
   │  Pin 6: M1 (motor +) ←── OUT1 + C7      │
   └──────────────────────────────────────────┘
```

### 4.3 Pin-by-Pin Reference (Every Pololu Pin Accounted For)

| Pololu Pin | Name | Connect To | Why |
|-----------|------|-----------|-----|
| **1** | VM | **Leave NC** | Motor supply AFTER reverse protection. Not needed; power enters via VIN. |
| **2** | GND | **Motor zone GND pour** | Ground for DRV8874 power stage |
| **3** | EN/IN1 | **Teensy PWM pin (2 or 5)** | Speed control. Apply 20 kHz PWM. Duty = speed. |
| **4** | PH/IN2 | **Teensy GPIO (3 or 6)** | Direction. HIGH = forward, LOW = reverse. |
| **5** | PMODE | **GND (motor zone pour)** | **CRITICAL.** Must be LOW to select PH/EN mode. If floating, wrong mode is latched! |
| **6** | SLEEP | **Teensy GPIO (29 or 30)** | Enable. Drive HIGH to wake (1ms delay). LOW = sleep. Defaults to sleep (100k internal pulldown). |
| **7** | VREF | **Leave NC** | 10k pullup to SLEEP on-board. VREF ≈ 3.3V. Sets current trip at 2.91A. No external connection needed. |
| **8** | VIN | **Battery VM (7.4V from J2)** | Motor power input, through reverse-voltage MOSFET. Add C4 (100uF) nearby. |
| **9** | GND | **Motor zone GND pour** | Second GND pin. Same net as pin 2. |
| **10** | OUT1 | **J3/J4 Pin 6 (M1)** | Motor terminal 1. Add C7 (100nF) to GND at connector. |
| **11** | OUT2 | **J3/J4 Pin 1 (M2)** | Motor terminal 2. Add C8 (10nF) to GND at connector. |
| **12** | IMODE | **Leave NC** | 20k pulldown on-board = cycle-by-cycle chopping + auto retry OCP. No external connection needed. |
| **13** | FAULT | **R4 (10k) to 3.3V + Teensy GPIO (33 or 34)** | Open-drain, active LOW. R4 pulls HIGH normally. Goes LOW on fault (OCP/UVLO/TSD). |
| **14** | CS | **R6 (1k) + C9 (10nF) filter -> Teensy ADC (A6 or A7)** | Current sense output. ~1.1 V/A. Filter: fc = 15.9 kHz for torque control inner loop. |

### 4.4 FAULT Pin Detail (Open-Drain Explained)

```
         3.3V
          │
       R4 (10k)       ← YOU add this resistor on main PCB
          │
          ├──────────── Teensy Pin 33 or 34 (digital input)
          │
   Pololu FAULT pin
          │
     ┌────┴────┐
     │ DRV8874 │
     │ internal│
     │         │
     │  open   │
     │  drain  │
     │ MOSFET  │─── GND     ← only closes during a fault
     └─────────┘

   Normal:  MOSFET open  → pin pulled to 3.3V by R4 → digitalRead() = HIGH
   Fault:   MOSFET closed → pin shorted to GND       → digitalRead() = LOW
```

### 4.5 CS Pin Detail (Current Sense for Torque Control)

```
   DRV8874 IPROPI pin (inside Pololu carrier)
          │
          │     ┌──────────────────────────────┐
          │     │ On Pololu carrier (built-in) │
          ├─────┤ 2.49k resistor to GND        │
          │     │ Converts current to voltage   │
          │     │ Output: ~1.1 V/A              │
          │     └──────────────────────────────┘
          │
   Pololu CS pin (header pin 14)
          │
          │     ┌──────────────────────────────┐
          │     │ YOU add on main PCB           │
          ├─────┤ R6 (1k) ──┬── Teensy ADC     │
          │     │            │   (A6 or A7)     │
          │     │       C9 (10nF)               │
          │     │            │                  │
          │     │           GND                 │
          │     │                               │
          │     │ Low-pass filter               │
          │     │ fc = 1/(2π × 1k × 10nF)      │
          │     │    = 15.9 kHz                 │
          │     └──────────────────────────────┘

   At 0.65A (rated):  CS = 0.72V
   At 1.40A (stall):  CS = 1.54V
   At 2.91A (trip):   CS = 3.30V (VREF limit kicks in)
```

### 4.6 Motor EMI Caps Detail (C7, C8 -- Why and Where)

Brushed DC motors generate broadband electrical noise from the commutator brush arcing. Every time a carbon brush breaks contact with a rotor winding, the collapsing magnetic field creates a voltage spike. This happens thousands of times per second.

**Without caps -- noise enters the PCB:**
```
  DRV8874 OUT1 ───── long trace ───── J3 Pin 6 ───── wire ───── Motor
                                                                  ⚡ SPARK
                                                                  │
                                                        Noise travels back through
                                                        the motor wire, into J3,
                                                        along the OUT1 trace, and
                                                        radiates into:
                                                          - IMU (corrupts pitch)
                                                          - CS ADC (noisy current)
                                                          - Encoder signals
                                                          - Ground plane
```

**With caps at the connector -- noise stopped at the board edge:**
```
  DRV8874 OUT1 ───── trace ─────┬──── J3 Pin 6 ───── wire ───── Motor
                                │                                 ⚡ SPARK
                           C7 (100nF)                             │
                                │                       Noise tries to travel back
                               GND                      but C7 shunts it to GND
                                                         HERE, at the connector,
                                                         before it reaches the PCB

  DRV8874 OUT2 ───── trace ─────┬──── J3 Pin 1 ───── wire ───── Motor
                                │                                 ⚡ SPARK
                           C8 (10nF)                              │
                                │                       C8 catches higher-frequency
                               GND                      noise that C7 misses
```

**Why two different values:**

Every capacitor has a self-resonant frequency (SRF) above which it stops working as a capacitor. Using two caps extends the effective filtering range:

| Cap | Value | Code on cap | Effective range | What it catches |
|-----|-------|-------------|----------------|-----------------|
| C7 | 100 nF | **104** | ~100 kHz to ~10 MHz | Brush commutation transients |
| C8 | 10 nF | **103** | ~1 MHz to ~100 MHz | Higher-frequency ringing |

Together they cover the full noise spectrum of a brushed DC motor.

**Placement rule:** C7 and C8 MUST be placed **at the JST-PH connector pins** (J3/J4), not near the DRV8874. The goal is to intercept noise at the point where motor wires enter the board.

```
  ┌─── MOTOR ZONE ──────────────────────────────────────────────┐
  │                                                              │
  │   [Pololu DRV8874]                     [J3 connector]       │
  │    OUT1 ─────── long trace ──────────┬── Pin 6 (M1) ── wire │──→ Motor
  │                                      │                      │
  │                                 C7 (100nF)  ← PLACE HERE    │
  │                                      │                      │
  │    OUT2 ─────── long trace ──────────┬── Pin 1 (M2) ── wire │──→ Motor
  │                                      │                      │
  │                                 C8 (10nF)   ← PLACE HERE    │
  │                                      │                      │
  │                                     GND (motor zone pour)   │
  └──────────────────────────────────────────────────────────────┘
```

> **Capacitor recommendation:** Standard ceramic disc or MLCC capacitors work. A bulk assortment
> kit with 100nF (code 104) and 10nF (code 103) in through-hole or 0805 SMD is all you need.
> X7R or X5R dielectric preferred (stable over temperature), but Y5V also works for EMI filtering.
> Voltage rating: any 16V or higher (we're at 7.4V motor supply).

### 4.7 Motor A vs Motor B Pin Assignments

| Signal | Motor A (Left Zone) | Motor B (Right Zone) |
|--------|--------------------|--------------------|
| EN/IN1 (PWM speed) | Teensy **Pin 2** (FlexPWM4) | Teensy **Pin 5** (FlexPWM2) |
| PH/IN2 (direction) | Teensy **Pin 3** | Teensy **Pin 6** |
| SLEEP (enable) | Teensy **Pin 29** | Teensy **Pin 30** |
| FAULT (fault flag) | Teensy **Pin 33** | Teensy **Pin 34** |
| CS (current sense) | Teensy **Pin 20** (A6) | Teensy **Pin 21** (A7) |
| OUT1 → M1 | **J3** Pin 6 | **J4** Pin 6 |
| OUT2 → M2 | **J3** Pin 1 | **J4** Pin 1 |
| Encoder A | Teensy **Pin 0** | Teensy **Pin 31** |
| Encoder B | Teensy **Pin 1** | Teensy **Pin 32** |
| VIN | Battery VM from J2 | Battery VM from J2 |
| PMODE | GND | GND |
| VM, VREF, IMODE | NC | NC |

### 4.8 PH/EN Mode Truth Table

We use PH/EN mode (PMODE = LOW). The EN pin receives PWM for speed, PH pin sets direction.

| SLEEP | EN (IN1) | PH (IN2) | OUT1 | OUT2 | Motor Action |
|-------|----------|----------|------|------|-------------|
| LOW | X | X | Hi-Z | Hi-Z | **Sleep** (low power, default state) |
| HIGH | 0 | X | LOW | LOW | **Brake** (slow decay, both outputs shorted to GND) |
| HIGH | PWM | HIGH | PWM | LOW | **Forward** at PWM duty % |
| HIGH | PWM | LOW | LOW | PWM | **Reverse** at PWM duty % |

- During the PWM LOW phase, both outputs go LOW (brake/slow decay). This keeps current flowing through the motor windings, giving smooth torque -- essential for balance control.
- Startup sequence: Set PMODE LOW (already tied to GND), then set SLEEP HIGH, wait 1 ms (tWAKE), then begin PWM.

### 4.9 Torque Control Architecture

For a brushed DC motor: **T = Kt x I** (torque = torque constant x current). Controlling current directly gives linear, predictable torque regardless of battery voltage or motor speed.

```
                   500-1000 Hz                      10-20 kHz
EKF2 pitch ──→ [Balance PID] ──→ I_desired ──→ [Current PI] ──→ PWM duty
                                                    ↑
                                                    │
                                              I_actual (CS pin ADC)
                                              read every PWM cycle
```

**Why this matters for a balancing robot:**
- The balance PID commands torque (Amps), not voltage. Same corrective force at 8.4V (full) or 6.0V (depleted).
- The inner current loop rejects back-EMF disturbances faster than the balance loop.
- Inherent current limiting -- motor can never exceed the commanded current.

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

#### PM02D I2C Routing (Pins 18/19)
- Route SCL and SDA as a pair, **away from PWM lines** and the SPI bus.
- Keep traces within the logic zone (no keep-out crossing needed).
- No external pull-ups needed (PM02D has on-board pull-ups).
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
3. **J1 (PM02D Molex connector)** -- place near Teensy I2C pins (18/19).
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
| PM02D power + I2C | PM02D module | J1 (JST-PH 6-pin) | Re-crimped cable | < 200 mm |
| Battery VM | PM02D XT60 output | J2 (XT30) | 18-20 AWG | < 150 mm |
| Motor A + Encoder A | Hiwonder motor | J3 (JST-PH 6-pin) | Stock cable | As short as possible |
| Motor B + Encoder B | Hiwonder motor | J4 (JST-PH 6-pin) | Stock cable | As short as possible |
| UART2 companion | RPi / BBB | J5 (JST-PH 4-pin) | 24-26 AWG | < 300 mm |

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

The system uses two nested control loops with different rates:

#### Inner Loop -- Current/Torque Control (10-20 kHz)

Runs inside the DRV8874 motor driver, synchronized to PWM frequency.

| Step | Time | Notes |
|------|------|-------|
| ADC read (CS pin) | ~2 us | Single channel, 12-bit |
| PI computation | ~1 us | Simple proportional + integral |
| PWM duty update | < 1 us | Register write |
| **Total per motor** | **~4 us** | Two motors = ~8 us per PWM cycle |

At 20 kHz PWM: 50 us per cycle, uses ~8 us = **16% CPU**. Comfortable margin.

#### Outer Loop -- Balance Control (500-1000 Hz)

Triggered by IMU data-ready interrupt.

| Step | Time | Notes |
|------|------|-------|
| SPI IMU read (14 bytes @ 7 MHz) | ~16 us | Accel + gyro registers |
| EKF2 attitude estimation | ~5-10 us | Quaternion update (PX4) |
| Balance PID computation | ~5 us | Pitch error -> torque command |
| Encoder read (hardware counter) | < 1 us | Optional, for position hold |
| Publish uORB topics | ~2 us | actuator_motors + balance_status |
| **Total** | **~30 us** | **Supports up to 30+ kHz loop rate** |

At 1 kHz: 1000 us per cycle, uses ~30 us = **3% CPU**. The vast majority of CPU time is available for the inner current loop, logging, and MAVLink telemetry.

#### PM02D Power Monitoring (10-50 Hz, separate bus)

| Step | Time | Bus |
|------|------|-----|
| INA226 voltage read | ~35 us | I2C @ 400 kHz (Pins 18/19) |
| INA226 current read | ~35 us | I2C @ 400 kHz |
| **Total** | **~70 us** | **Completely independent of SPI/ADC** |

The I2C power monitor runs on a low-priority work queue and a separate bus. It cannot interfere with either control loop.

#### Key Timing Notes

- The SPI upgrade from I2C reduces IMU read time from ~350 us to ~16 us, critical for high-rate balance control.
- Use the IMU's data-ready interrupt (Pin 17) to trigger outer loop reads at the sensor's output data rate (up to 1.125 kHz for accel/gyro on ICM-20948).
- The inner current loop runs from a FlexPWM timer interrupt, synchronized to the PWM cycle. ADC sampling is triggered at the PWM midpoint (center of the ON period) for cleanest current measurement.
- The CS pin R6/C9 filter (fc = 15.9 kHz) is designed to pass the inner loop bandwidth while filtering PWM switching transients.

---

## 6. Firmware Initialization Reference

```cpp
#include <SPI.h>

// === Pin Definitions ===

// IMU (SPI0)
#define IMU_CS      10    // ICM-20948 SPI chip select (SPI0 CS0)
#define IMU_INT     17    // ICM-20948 data-ready interrupt
// SPI0 hardware pins: SCK=13, MOSI=11, MISO=12

// PM02D Power Sensing (I2C, no pin defines needed -- handled by INA226 driver)
// PM02D INA226 on I2C0 (Wire): SCL = Pin 19, SDA = Pin 18, address 0x40

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

## 7. Schematic Review Status

### Verified Correct (all 23 Teensy pin assignments match design guide)

| Signal | Teensy Pin | Schematic Label | Status |
|--------|-----------|-----------------|--------|
| Encoder A phase A | Pin 0 | ENC_A_PH_A | OK |
| Encoder A phase B | Pin 1 | ENC_A_PH_B | OK |
| Motor A PWM | Pin 2 | MOTA_PWM | OK |
| Motor A direction | Pin 3 | MOTA_DIR | OK |
| Motor B PWM | Pin 5 | MOTB_PWM | OK |
| Motor B direction | Pin 6 | MOTB_DIR | OK |
| UART2 RX (companion) | Pin 7 | UART_COMP_RX | OK |
| UART2 TX (companion) | Pin 8 | UART_COMP_TX | OK |
| IMU SPI CS | Pin 10 | ICM20948_CS | OK |
| IMU SPI MOSI | Pin 11 | ICM20948_SDI | OK |
| IMU SPI MISO | Pin 12 | ICM20948_SDO | OK |
| IMU SPI SCK | Pin 13 | ICM20948_SCL | OK |
| IMU Interrupt | Pin 17 | ICM20948_INT | OK |
| PM02D I2C SDA | Pin 18 | PM02_SDA | OK |
| PM02D I2C SCL | Pin 19 | PM02_SCL | OK |
| Motor A current sense | Pin 20 (A6) | MOTA_CUR | OK |
| Motor B current sense | Pin 21 (A7) | MOTB_CUR | OK |
| Motor A SLEEP | Pin 29 | MOTA_SLEEP | OK |
| Motor B SLEEP | Pin 30 | MOTB_SLEEP | OK |
| Encoder B phase A | Pin 31 | ENC_B_PH_A | OK |
| Encoder B phase B | Pin 32 | ENC_B_PH_B | OK |
| Motor A FAULT | Pin 33 | MOTA_FAULT | OK |
| Motor B FAULT | Pin 34 | MOTB_FAULT | OK |

### Schematic Component Summary (as-built)

| Ref | Component | Value | Footprint | Status |
|-----|-----------|-------|-----------|--------|
| U1 | Teensy 4.1 | -- | Teensy41:Teensy41 | OK |
| U2 | ICM-20948 Carrier | -- | Adafruit_ICM20948:ICM20948_Carrier | OK |
| U3 | DRV8874 Carrier (Motor A) | -- | Pololu_DRV8874:DRV8874_Carrier | OK |
| U4 | DRV8874 Carrier (Motor B) | -- | Pololu_DRV8874:DRV8874_Carrier | OK |
| J1 | PM02D connector | 6-pin | JST_PH_S6B-PH-K | OK |
| J2 | UART companion | 4-pin | JST_PH_S4B-PH-K | OK |
| J3 | Battery XT30 | 2-pin | AMASS_XT30U-F | OK |
| J4 | Motor A + Encoder A | 6-pin | JST_PH_S6B-PH-K | OK |
| J5 | Motor B + Encoder B | 6-pin | JST_PH_S6B-PH-K | OK |
| C1 | VIN bypass cap | 100nF | C_Disc_D5.0mm | OK |
| C2 | CS filter cap (Motor A) | 10nF | C_Disc_D5.0mm | OK |
| C3 | Motor A EMI cap (OUT1) | 100nF | C_Disc_D5.0mm | OK |
| C4 | Motor A EMI cap (OUT2) | 10nF | C_Disc_D5.0mm | OK |
| C5 | Motor B EMI cap (OUT1) | 100nF | C_Disc_D5.0mm | OK |
| C6 | Motor B EMI cap (OUT2) | 10nF | C_Disc_D5.0mm | OK |
| R1 | FAULT pullup Motor A | 10k | R_Axial_DIN0204 | OK |
| R2 | CS filter Motor A | 1k | R_Axial_DIN0204 | OK |
| R3 | FAULT pullup Motor B | 10k | R_Axial_DIN0204 | OK |
| R4 | CS filter Motor B | 1k | R_Axial_DIN0204 | OK |

### Issues to Fix

**1. MISSING: 10uF bulk cap on Teensy VIN**
The design guide calls for C1 (100nF) + a 10uF bulk cap on Teensy VIN. Only C1 (100nF) is placed. Add a 10uF electrolytic or ceramic cap between +5V and GND near the Teensy.
- Footprint: `Capacitor_THT:CP_Radial_D5.0mm_P2.50mm` (electrolytic) or `C_Disc_D5.0mm` (ceramic if available in 10uF)

**2. MISSING: 100uF bulk caps near DRV8874 VIN (x2)**
Each Pololu carrier needs a 100uF 16V electrolytic near its VIN pin to handle motor current transients. Neither is in the schematic.
- Footprint: `Capacitor_THT:CP_Radial_D5.0mm_P2.50mm`
- One per motor zone, placed within 10mm of each Pololu VIN pin.

**3. MISSING: CS filter cap for Motor B**
C2 (10nF) is the CS filter cap for Motor A (pairs with R2 1k). Motor B has R4 (1k CS filter resistor) but no matching 10nF filter cap. Add one.

**4. Battery power symbol is +12V -- should be VBAT or +7V4**
The schematic uses `power:+12V` for the battery VM rail, but the actual voltage is 7.4V (2S LiPo, range 6.0-8.4V). This is misleading. Consider:
- Replace with a net label `VBAT` or `VM`
- Or create a custom power symbol `+7V4`
- Not a functional error, but confusing during review.

**5. 49 no-connect flags on unused Teensy pins -- OK**
Correct count for the ~49 unused Teensy pins. No issue.

**6. 3 PWR_FLAG symbols -- verify placement**
PWR_FLAGs are needed on power nets that are driven by connectors (not by IC power outputs). Verify they are on: +5V (from J1 PM02D), GND, and +12V/VBAT (from J3 XT30).

### Schematic Checklist (updated with actual ref designators)

#### Logic Zone
- [x] U1 (Teensy 4.1) VIN connected to PM02D VCC via J1 with C1 (100nF)
- [ ] **ADD: 10uF bulk cap on Teensy VIN (C1 alone is insufficient)**
- [x] U2 (ICM-20948) VIN connected to +3V3
- [x] SPI0: ICM20948_SCL(Pin13), ICM20948_SDI(Pin11), ICM20948_SDO(Pin12), ICM20948_CS(Pin10)
- [x] ICM20948_INT connected to Teensy Pin 17
- [x] J1 (PM02D): Pin 3 SCL -> PM02_SCL -> Teensy Pin 19; Pin 4 SDA -> PM02_SDA -> Teensy Pin 18
- [x] J2 (UART): Pin 1 TX=Pin 8, Pin 2 RX=Pin 7, Pin 3 GND, Pin 4 3V3
- [x] 49 no-connect flags on unused Teensy pins

#### Motor Zone A (U3)
- [x] U3 EN/IN1 <- MOTA_PWM <- Teensy Pin 2
- [x] U3 PH/IN2 <- MOTA_DIR <- Teensy Pin 3
- [x] U3 SLEEP <- MOTA_SLEEP <- Teensy Pin 29
- [x] U3 FAULT -> R1 (10k to 3.3V) -> MOTA_FAULT -> Teensy Pin 33
- [x] U3 CS -> R2 (1k) + C2 (10nF) -> MOTA_CUR -> Teensy Pin 20 (A6)
- [x] U3 OUT1 -> C3 (100nF to GND) -> J4 Pin 6 (M1)
- [x] U3 OUT2 -> C4 (10nF to GND) -> J4 Pin 1 (M2)
- [x] U3 PMODE tied to GND
- [x] U3 VM, VREF, IMODE left NC
- [x] J4 Pin 2 (encoder V) <- 3.3V; Pin 3 (A) -> ENC_A_PH_A -> Pin 0; Pin 4 (B) -> ENC_A_PH_B -> Pin 1
- [ ] **ADD: 100uF bulk cap near U3 VIN**

#### Motor Zone B (U4)
- [x] U4 EN/IN1 <- MOTB_PWM <- Teensy Pin 5
- [x] U4 PH/IN2 <- MOTB_DIR <- Teensy Pin 6
- [x] U4 SLEEP <- MOTB_SLEEP <- Teensy Pin 30
- [x] U4 FAULT -> R3 (10k to 3.3V) -> MOTB_FAULT -> Teensy Pin 34
- [x] U4 CS -> R4 (1k) -> MOTB_CUR -> Teensy Pin 21 (A7)
- [ ] **ADD: 10nF CS filter cap for Motor B (R4 has no matching cap)**
- [x] U4 OUT1 -> C5 (100nF to GND) -> J5 Pin 6 (M1)
- [x] U4 OUT2 -> C6 (10nF to GND) -> J5 Pin 1 (M2)
- [x] U4 PMODE tied to GND
- [x] U4 VM, VREF, IMODE left NC
- [x] J5 Pin 2 (encoder V) <- 3.3V; Pin 3 (A) -> ENC_B_PH_A -> Pin 31; Pin 4 (B) -> ENC_B_PH_B -> Pin 32
- [ ] **ADD: 100uF bulk cap near U4 VIN**

#### Power Rails
- [ ] **FIX: +12V power symbol should be VBAT or +7V4 (battery is 7.4V, not 12V)**
- [x] +5V rail from J1 PM02D to Teensy VIN
- [x] +3V3 rail from Teensy to IMU, encoders, FAULT pullups
- [x] GND symbols placed throughout
- [x] PWR_FLAG symbols present (3x)

### Layout Checklist (for PCB phase)

#### Zoned Ground Plane
- [ ] Three separate ground pours: Logic (center), Motor A (left), Motor B (right)
- [ ] Keep-out areas (2-3 mm gap) between logic pour and each motor pour
- [ ] GND bridge trace (1.0 mm) at each keep-out crossing, adjacent to signal group
- [ ] No traces routed under IMU footprint on ground plane layer
- [ ] VM copper pour from J3 (XT30) to both motor zones (1.5 mm min or pour)

#### Self-Balancing
- [ ] IMU placed at board center, aligned above wheel axle when mounted
- [ ] IMU X-axis aligned with robot forward/backward (pitch) direction
- [ ] Solid unbroken logic ground pour under IMU footprint
- [ ] Board is left-right symmetric (Motor A zone mirrors Motor B zone)
- [ ] J4 on left edge, J5 on right edge (symmetric motor cable routing)
- [ ] 4x M3 mounting holes, symmetric, with space for rubber grommets

#### Mechanical / Chassis Integration
- [ ] Rubber grommets or soft-mount standoffs between PCB and chassis
- [ ] Battery mounted above wheel axle (high CoG for slower pendulum dynamics)
- [ ] All cables strain-relieved to chassis (not to PCB)
- [ ] Motor cables routed symmetrically left-to-right
- [ ] Hiwonder PH2.0 motor cables plug directly into J4/J5

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
