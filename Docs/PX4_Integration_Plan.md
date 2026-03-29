# PX4 Integration Plan -- Self-Balancing Robot on Custom Teensy 4.1 Board

## Overview

This document describes the plan to run **PX4 Autopilot** on a custom single-PCB balancing robot using:
- **MCU:** Teensy 4.1 (NXP i.MX RT1062, Cortex-M7 @ 600 MHz)
- **IMU:** Adafruit ICM-20948 breakout (9-DoF: accel + gyro + AK09916 mag) over SPI
- **Motor Drivers:** 2x Pololu DRV8874 carrier boards (PH/EN mode)
- **Motors:** 2x Hiwonder JGA27-310R20 DC gear motors with Hall encoders
- **Power:** Holybro PM02 power module (5V regulated + battery pass-through)

PX4 is used as a **HAL and middleware framework** (not as a flight controller). We leverage:
- uORB pub/sub message bus
- Existing ICM-20948 SPI driver + AK09916 magnetometer driver
- EKF2 attitude estimator (provides pitch angle for free)
- Parameter system (runtime PID tuning via QGroundControl)
- uLog data logging to SD card
- MAVLink telemetry and monitoring
- Work queue scheduler for deterministic control loop timing

---

## 1. PX4 ICM-20948 Driver Status

**Driver exists and is production-ready.**

- Location: `src/drivers/imu/invensense/icm20948/`
- Interface: **SPI (default)**, 7 MHz clock speed
- Files:
  - `ICM20948.cpp` / `.hpp` -- main accel+gyro driver
  - `ICM20948_AK09916.cpp` / `.hpp` -- integrated AK09916 magnetometer
  - `ICM20948_I2C_Passthrough.cpp` -- I2C passthrough variant (not needed)
  - `icm20948_main.cpp` -- SPI entry point, initializes with SPI enabled by default
  - `InvenSense_ICM20948_registers.hpp` -- full register map
  - `AKM_AK09916_registers.hpp` -- magnetometer register map
- Startup command: `icm20948 -s -b <SPI_bus> -R <rotation> -M start`
  - `-s` = SPI mode
  - `-b <N>` = SPI bus number
  - `-R <N>` = rotation enum (board orientation)
  - `-M` = enable on-chip AK09916 magnetometer
- Publishes to uORB: `sensor_accel`, `sensor_gyro`, `sensor_mag`

Part of the broader InvenSense driver family (`src/drivers/imu/invensense/`) which covers 16+ chip variants sharing common infrastructure.

**The Tropic Community VMU board config already enables ICM20948** in its `default.px4board`, confirming it builds and works on the i.MX RT1062 / Teensy 4.1 platform.

---

## 2. Starting Point: Tropic Community VMU Board Target

The `nxp/tropic-community` board target in PX4 (`boards/nxp/tropic-community/`) runs on the Teensy 4.1 and is the base for our custom board target.

**Key facts:**
- Build command: `make nxp_tropic-community`
- Flash: `make nxp_tropic-community upload_teensy` (uses `teensy-loader-cli`)
- Output: `.hex` file flashed via Teensy bootloader
- Confirmed working on bare Teensy 4.1 (boots, provides MAVLink over USB)
- MCU: NXP MIMXRT1062DVJ6B (196-pin BGA, same as Teensy 4.1)
- Board ID: 36

**What it enables that we'll keep:**
- ICM20948 driver (already in build config)
- EKF2 attitude estimator
- uORB, MAVLink, uLog, parameter system
- SD card logging (via Teensy 4.1 SD slot)
- USB MAVLink console

**What it enables that we'll disable:**
- BMI088, ICM42688P, ICM45686 IMU drivers (not on our board)
- BMM150 external magnetometer (we use ICM-20948's internal AK09916)
- BMP388 barometer (not needed for ground robot)
- INA226/228/238 power monitors (we use PM02 analog or PM02D I2C)
- GPS, CAN, telemetry UART (not needed initially)
- Navigator, geofence, mission modules (not needed for balance bot)
- Fixed-wing, VTOL controllers (not needed)

---

## 3. Custom Board Target Structure

Create a new board target by forking `nxp/tropic-community`:

```
boards/custom/balancebot/
├── default.px4board            # Build config: which drivers and modules to include
├── firmware.prototype          # Board metadata (board_id, etc.)
├── cmake/
│   └── upload.cmake            # Flash script (teensy-loader-cli)
├── init/
│   ├── rc.board_sensors        # Sensor startup script
│   ├── rc.board_defaults       # Default parameters for balance bot
│   └── rc.board_extras         # Optional extra startup
├── nuttx-config/
│   ├── include/
│   │   └── board.h             # Pin multiplexing, clock config
│   ├── nsh/
│   │   └── defconfig           # NuttX kernel configuration
│   └── scripts/
│       └── flash.ld            # Linker script (from Tropic, unchanged)
└── src/
    ├── board_config.h          # SPI bus, CS GPIO, PWM timer assignments
    ├── init.c                  # Board initialization code
    ├── spi.cpp                 # SPI bus configuration
    ├── timer_config.cpp        # PWM output timer configuration
    └── manifest.c              # Board manifest
```

---

## 4. Pin Mapping: Custom Board -> PX4 Config

### 4.1 Hardware Pin Assignments (from Schematic Design Guide)

| Teensy Pin | Function | PX4 Mapping |
|------------|----------|-------------|
| **13** | SPI0 SCK -> ICM-20948 SCL | LPSPI4_SCK (or whichever NuttX SPI bus maps here) |
| **11** | SPI0 MOSI -> ICM-20948 SDA | LPSPI4_SDO |
| **12** | SPI0 MISO -> ICM-20948 SDO | LPSPI4_SDI |
| **10** | SPI0 CS -> ICM-20948 CS | GPIO chip-select, active low |
| **17** | ICM-20948 INT (data-ready) | DRDY interrupt GPIO |
| **2** | PWM -> Pololu A: EN/IN1 | FlexPWM4_PWMA02, Motor A speed |
| **3** | GPIO -> Pololu A: PH/IN2 | GPIO output, Motor A direction |
| **29** | GPIO -> Pololu A: SLEEP | GPIO output, Motor A enable |
| **5** | PWM -> Pololu B: EN/IN1 | FlexPWM2_PWMA01, Motor B speed |
| **6** | GPIO -> Pololu B: PH/IN2 | GPIO output, Motor B direction |
| **30** | GPIO -> Pololu B: SLEEP | GPIO output, Motor B enable |
| **0** | Encoder A, channel A | Quadrature decoder or GPIO interrupt |
| **1** | Encoder A, channel B | Quadrature decoder or GPIO interrupt |
| **31** | Encoder B, channel A | GPIO interrupt |
| **32** | Encoder B, channel B | GPIO interrupt |
| **20 (A6)** | Pololu A: CS (current sense) | ADC input |
| **21 (A7)** | Pololu B: CS (current sense) | ADC input |
| **33** | Pololu A: FAULT | GPIO input (active low, ext pullup) |
| **34** | Pololu B: FAULT | GPIO input (active low, ext pullup) |
| **14 (A0)** | PM02 V_SENSE | ADC input |
| **15 (A1)** | PM02 I_SENSE | ADC input |

### 4.2 Teensy 4.1 Pin -> i.MX RT1062 Pad Mapping

The Teensy 4.1 uses the MIMXRT1062DVJ6B. PX4/NuttX needs the actual i.MX RT pad names, not Arduino pin numbers. Key mappings:

| Teensy Pin | i.MX RT Pad | GPIO | Alt Function |
|------------|-------------|------|-------------|
| 10 (CS) | AD_B0_00 | GPIO1_IO00 | LPSPI1_PCS0 (or GPIO for manual CS) |
| 11 (MOSI) | AD_B0_02 | GPIO1_IO02 | LPSPI1_SDO |
| 12 (MISO) | AD_B0_01 | GPIO1_IO01 | LPSPI1_SDI |
| 13 (SCK) | AD_B0_03 | GPIO1_IO03 | LPSPI1_SCK |
| 2 (PWM_A) | EMC_04 | GPIO4_IO04 | FLEXPWM4_PWMA02 |
| 5 (PWM_B) | EMC_08 | GPIO4_IO08 | FLEXPWM2_PWMA01 |

> **Note:** The Teensy 4.1 SPI0 maps to **LPSPI1** (not LPSPI4) on the i.MX RT1062.
> The Arduino SPI library abstracts this, but NuttX/PX4 needs the actual peripheral name.
> Verify all pad mappings against the Teensy 4.1 schematic and i.MX RT1062 reference manual
> before configuring `board.h`. The PJRC "Teensy 4.1 Pins Mux" document in the Docs folder
> is the primary reference for these mappings.

---

## 5. Pin Configuration Chain -- How PX4 Finds the ICM-20948

PX4 uses **4 files** to map physical pins to sensor drivers. Each file operates at a different layer. Understanding this chain is critical for porting to a new board.

```
Layer 1: board.h           → Pin mux:  "Pad GPIO_AD_B0_00 is LPSPI1_SCK"
Layer 2: spi.cpp           → Device map: "LPSPI1 has ICM-20948, CS = GPIO1_IO03"
Layer 3: board_config.h    → Support:   "LPSPI1 off-state pins, DRDY interrupt GPIO"
Layer 4: rc.board_sensors  → Runtime:   "icm20948 -s -b 1 -M start"
```

### 5.0.1 Layer 1: `nuttx-config/include/board.h` -- Pin Mux (NuttX Level)

This tells NuttX which physical i.MX RT pad maps to which peripheral function. It does NOT mention any sensor by name -- it just configures the SPI peripheral pins.

**Tropic VMU (reference -- LPSPI3 for ICM-42688P):**
```c
#define GPIO_LPSPI3_SCK   GPIO_LPSPI3_SCK_1   // GPIO_AD_B1_15
#define GPIO_LPSPI3_SDI   GPIO_LPSPI3_SDI_1   // GPIO_AD_B1_13 (MISO)
#define GPIO_LPSPI3_SDO   GPIO_LPSPI3_SDO_1   // GPIO_AD_B1_14 (MOSI)
```

**Your board (LPSPI1 for ICM-20948 on Teensy SPI0):**
```c
#define GPIO_LPSPI1_SCK   GPIO_LPSPI1_SCK_3   // GPIO_AD_B0_00 → Teensy Pin 13
#define GPIO_LPSPI1_SDI   GPIO_LPSPI1_SDI_3   // GPIO_AD_B0_01 → Teensy Pin 12 (MISO)
#define GPIO_LPSPI1_SDO   GPIO_LPSPI1_SDO_3   // GPIO_AD_B0_02 → Teensy Pin 11 (MOSI)
```

> **Note:** The `_3` suffix selects the pad mux alternate function. The i.MX RT1062 has
> multiple possible pad assignments for each SPI peripheral. The correct suffix must be
> looked up in the i.MX RT1062 reference manual (Chapter 11, IOMUXC) cross-referenced
> with the Teensy 4.1 pin mux table (`Docs/Teensy4.1 Pins Mux.pdf`).

### 5.0.2 Layer 2: `src/spi.cpp` -- SPI Bus Device Map (PX4 Level)

This is the **critical** file. It tells PX4: "On this SPI bus, there is this specific sensor, and here is its chip-select GPIO." When the ICM-20948 driver starts, it looks up this table to find which CS pin to assert.

**Tropic VMU (reference):**
```cpp
constexpr px4_spi_bus_t px4_spi_buses[SPI_BUS_MAX_BUS_ITEMS] = {
    initSPIBus(SPI::Bus::LPSPI3, {
        initSPIDevice(DRV_IMU_DEVTYPE_ICM42688P, SPI::CS{GPIO::Port1, GPIO::Pin28}),
    }),
    initSPIBus(SPI::Bus::LPSPI4, {
        initSPIDevice(DRV_GYR_DEVTYPE_BMI088, SPI::CS{GPIO::Port2, GPIO::Pin18}),
        initSPIDevice(DRV_ACC_DEVTYPE_BMI088, SPI::CS{GPIO::Port2, GPIO::Pin0}),
    }),
};
```

**Your board:**
```cpp
// ICM-20948 on LPSPI1, CS = Teensy Pin 10 = GPIO_AD_B0_03 = GPIO1_IO03
constexpr px4_spi_bus_t px4_spi_buses[SPI_BUS_MAX_BUS_ITEMS] = {
    initSPIBus(SPI::Bus::LPSPI1, {
        initSPIDevice(DRV_IMU_DEVTYPE_ICM20948, SPI::CS{GPIO::Port1, GPIO::Pin3}),
    }),
};
```

> **How to find the GPIO port and pin:**
> Teensy Pin 10 = i.MX RT pad `GPIO_AD_B0_03`.
> AD_B0 pads map to GPIO1. Pad 03 = Pin 3.
> Therefore: `GPIO::Port1, GPIO::Pin3`.
> See i.MX RT1062 reference manual, Chapter 12 (GPIO) for the full pad-to-GPIO mapping.

### 5.0.3 Layer 3: `src/board_config.h` -- Board Support Defines

Defines SPI pin OFF states (for power-down tri-stating), DRDY interrupt GPIO, ADC channels, and other board-level constants.

**Your board additions:**
```cpp
// LPSPI1 off-state pins (tri-state when sensor is powered down)
#define GPIO_LPSPI1_SCK_OFF  _PIN_OFF(GPIO_AD_B0_00)  // Teensy Pin 13
#define GPIO_LPSPI1_MISO_OFF _PIN_OFF(GPIO_AD_B0_01)  // Teensy Pin 12
#define GPIO_LPSPI1_MOSI_OFF _PIN_OFF(GPIO_AD_B0_02)  // Teensy Pin 11

// ICM-20948 data-ready interrupt (Teensy Pin 17 = GPIO_AD_B1_06 = GPIO1_IO22)
#define GPIO_ICM20948_DRDY   (GPIO_PORT1 | GPIO_PIN22 | GPIO_INPUT | IOMUX_PULL_DOWN)
```

### 5.0.4 Layer 4: `init/rc.board_sensors` -- Runtime Startup Script

This shell script runs at boot and actually starts the driver, telling it which bus number and mode to use.

**Your board:**
```bash
#!/bin/sh
# Start ICM-20948 on LPSPI1 (bus 1) in SPI mode with AK09916 magnetometer
icm20948 -s -b 1 -R 0 -M start
#   -s     SPI mode (not I2C)
#   -b 1   SPI bus number 1 = LPSPI1
#   -R 0   Rotation enum 0 (adjust for your IMU X-axis orientation on PCB)
#   -M     Enable on-chip AK09916 magnetometer
```

### 5.0.5 Runtime Sequence -- How It All Connects

```
1. NuttX boots
   → board.h muxes Teensy Pins 11/12/13 to LPSPI1 peripheral function

2. PX4 initializes
   → spi.cpp registers the device table:
     "LPSPI1 has ICM-20948, CS = GPIO1 Pin 3 (Teensy Pin 10)"

3. rc.board_sensors runs
   → Executes: icm20948 -s -b 1 -R 0 -M start

4. ICM-20948 driver starts:
   a. Looks up bus 1 in spi.cpp table → finds DRV_IMU_DEVTYPE_ICM20948
   b. Gets CS GPIO from the table → GPIO1_IO03 (Teensy Pin 10)
   c. Asserts CS LOW → sends SPI read of WHO_AM_I register
   d. Gets 0xEA back → confirms ICM-20948 is present
   e. Configures accel/gyro ODR, DLPF, scale
   f. Starts publishing to uORB: sensor_accel, sensor_gyro
   g. (-M flag) Initializes internal AK09916 magnetometer
   h. Starts publishing to uORB: sensor_mag
   i. Registers DRDY interrupt for precise sample timing

5. EKF2 subscribes to sensor_accel + sensor_gyro + sensor_mag
   → Publishes vehicle_attitude (quaternion, Euler angles, rates)

6. Your balance_control module subscribes to vehicle_attitude
   → Extracts pitch angle → runs PID → publishes actuator_motors
```

> **Key insight:** The ICM-20948 driver code itself contains ZERO pin information.
> It just asks PX4: "give me SPI bus 1." The board config files handle the rest.
> This is why the same driver binary works on completely different hardware --
> only the 4 config files change per board.

---

## 6. Board Config Files -- Key Changes

### 6.1 `default.px4board`

```cmake
# IMU -- only ICM-20948
CONFIG_DRIVERS_IMU_INVENSENSE_ICM20948=y
# Disable unused IMUs
# CONFIG_DRIVERS_IMU_INVENSENSE_ICM42688P is not set
# CONFIG_DRIVERS_IMU_BOSCH_BMI088 is not set

# Magnetometer -- via ICM-20948 internal AK09916
# No external mag driver needed

# Barometer -- not needed for ground robot
# CONFIG_DRIVERS_BAROMETER_BMP388 is not set

# Attitude estimation
CONFIG_MODULES_EKF2=y

# Actuators -- PWM for DRV8874
CONFIG_DRIVERS_PWM_OUT=y

# Keep these core modules
CONFIG_MODULES_COMMANDER=y
CONFIG_MODULES_MAVLINK=y
CONFIG_MODULES_LOGGER=y
CONFIG_MODULES_DATAMAN=y

# Custom balance controller module
CONFIG_MODULES_BALANCE_CONTROL=y

# Disable flight-specific modules
# CONFIG_MODULES_NAVIGATOR is not set
# CONFIG_MODULES_FW_ATT_CONTROL is not set
# CONFIG_MODULES_FW_POS_CONTROL is not set
# CONFIG_MODULES_VTOL_ATT_CONTROL is not set
# CONFIG_MODULES_MC_ATT_CONTROL is not set      # We use our own balance controller
# CONFIG_MODULES_MC_POS_CONTROL is not set
```

### 6.2 `init/rc.board_sensors`

```bash
#!/bin/sh
# Start ICM-20948 on SPI bus with magnetometer enabled
# -s = SPI mode
# -b 1 = LPSPI1 (Teensy SPI0 = i.MX RT LPSPI1)
# -R 0 = rotation (adjust based on IMU orientation on PCB)
# -M = enable AK09916 magnetometer
icm20948 -s -b 1 -R 0 -M start
```

### 6.3 `init/rc.board_defaults`

```bash
#!/bin/sh
# EKF2 configuration for ground robot
param set-default EKF2_MAG_TYPE 1          # Use magnetometer
param set-default EKF2_BARO_CTRL 0         # Disable barometer fusion (no baro)
param set-default EKF2_GPS_CTRL 0          # Disable GPS fusion (no GPS)
param set-default EKF2_HGT_REF 0           # No height reference needed

# Balance controller default gains (tune via QGroundControl)
param set-default BAL_PITCH_P 15.0
param set-default BAL_PITCH_I 0.5
param set-default BAL_PITCH_D 1.0
param set-default BAL_PITCH_SP 0.0         # Target pitch angle (upright)
param set-default BAL_MAX_PWM 0.8          # Max PWM duty (0-1)
param set-default BAL_LOOP_RATE 500        # Control loop Hz

# Motor configuration
param set-default BAL_PWM_FREQ 20000       # 20 kHz PWM
```

### 6.4 `src/board_config.h`

```cpp
#pragma once

// SPI bus for ICM-20948 (Teensy SPI0 = LPSPI1)
#define PX4_SPI_BUS_SENSORS     1
#define PX4_SPIDEV_ICM_20948    PX4_MK_SPI_SEL(PX4_SPI_BUS_SENSORS, 0)

// Chip select GPIO (Teensy Pin 10 = GPIO_AD_B0_00)
#define GPIO_SPI1_CS_ICM20948   /* GPIO_AD_B0_00, configure as GPIO output */

// ICM-20948 DRDY interrupt (Teensy Pin 17)
#define GPIO_ICM20948_DRDY      /* GPIO_AD_B1_06, configure as GPIO input, rising edge */

// Motor A (left) -- Pololu DRV8874 carrier A
#define GPIO_MOTOR_A_PWM        /* Teensy Pin 2:  FLEXPWM4_PWMA02 */
#define GPIO_MOTOR_A_DIR        /* Teensy Pin 3:  GPIO output */
#define GPIO_MOTOR_A_SLEEP      /* Teensy Pin 29: GPIO output */
#define GPIO_MOTOR_A_FAULT      /* Teensy Pin 33: GPIO input */
#define ADC_MOTOR_A_CURRENT     /* Teensy Pin 20 (A6): ADC channel */

// Motor B (right) -- Pololu DRV8874 carrier B
#define GPIO_MOTOR_B_PWM        /* Teensy Pin 5:  FLEXPWM2_PWMA01 */
#define GPIO_MOTOR_B_DIR        /* Teensy Pin 6:  GPIO output */
#define GPIO_MOTOR_B_SLEEP      /* Teensy Pin 30: GPIO output */
#define GPIO_MOTOR_B_FAULT      /* Teensy Pin 34: GPIO input */
#define ADC_MOTOR_B_CURRENT     /* Teensy Pin 21 (A7): ADC channel */

// Encoder A (Teensy Pins 0, 1)
#define GPIO_ENC_A_CHA          /* Teensy Pin 0: GPIO interrupt */
#define GPIO_ENC_A_CHB          /* Teensy Pin 1: GPIO interrupt */

// Encoder B (Teensy Pins 31, 32)
#define GPIO_ENC_B_CHA          /* Teensy Pin 31: GPIO interrupt */
#define GPIO_ENC_B_CHB          /* Teensy Pin 32: GPIO interrupt */

// PM02 power monitor (analog)
#define ADC_PM02_VSENSE         /* Teensy Pin 14 (A0): ADC channel */
#define ADC_PM02_ISENSE         /* Teensy Pin 15 (A1): ADC channel */
```

> **Note:** The actual GPIO pad names and register values must be filled in from the
> i.MX RT1062 reference manual and Teensy 4.1 pin mux table. The comments above show
> the Teensy pin number for reference. The Tropic Community VMU `board_config.h` and
> `board.h` are the best templates for the correct NuttX GPIO configuration macros.

---

## 7. Custom PX4 Modules to Write

### 7.0 Cascaded Control Architecture Overview

The system uses a **cascaded (two-loop) control** architecture for torque-based motor control:

```
                   Outer Loop (500-1000 Hz)                Inner Loop (10-20 kHz)

EKF2 pitch ──→ [Balance PID] ──→ I_desired (Amps) ──→ [Current PI] ──→ PWM duty
                    |                                        ↑
                    |                                        |
                    └── pitch error                   I_actual (CS pin)
                                                      ADC read per PWM cycle
```

**Why torque control (not voltage/PWM control):**
- For a brushed DC motor: **T = Kt x I** (torque = torque constant x current)
- Controlling current = controlling torque directly
- The robot applies the **same corrective force** whether battery is full (8.4V) or depleted (6.0V)
- The inner current loop rejects back-EMF disturbances (motor speed changes) faster than the balance loop can see them
- Inherent current limiting -- motor can never exceed commanded current, protecting DRV8874 and motor

**Loop separation:**

| Loop | Rate | Module | Input | Output |
|------|------|--------|-------|--------|
| Outer (balance) | 500-1000 Hz | `balance_control` | Pitch angle (EKF2) | Desired current (Amps) |
| Inner (current) | 10-20 kHz | `drv8874` driver | Desired current + CS ADC | PWM duty cycle |

### 7.1 Balance Controller Module (Outer Loop)

**Location:** `src/modules/balance_control/`

**Purpose:** Pitch stabilization. Reads attitude, outputs desired motor torque as a current command (Amps).

**uORB subscriptions (input):**
| Topic | Source | Data Used |
|-------|--------|-----------|
| `vehicle_attitude` | EKF2 | Quaternion -> pitch angle and pitch rate |
| `sensor_accel` | ICM-20948 driver | Raw accel (backup / high-rate) |
| `sensor_gyro` | ICM-20948 driver | Raw gyro (backup / high-rate) |
| `parameter_update` | Parameter system | PID gain changes from QGroundControl |

**uORB publications (output):**
| Topic | Destination | Data Published |
|-------|-------------|----------------|
| `actuator_motors` | DRV8874 driver | Current commands (Amps) per channel, range -I_max to +I_max |
| `balance_status` | Logger / MAVLink | Pitch angle, pitch rate, torque cmd, state |

**Control loop (runs at 500-1000 Hz via work queue):**

```cpp
void BalanceControl::Run()
{
    // 1. Get attitude from EKF2
    vehicle_attitude_s att;
    if (_att_sub.update(&att)) {
        // Extract pitch from quaternion
        matrix::Eulerf euler(matrix::Quatf(att.q));
        float pitch = euler.theta();
        float pitch_rate = att.pitchspeed;  // from gyro

        // 2. PID controller -- outputs desired current (Amps), not PWM duty
        float error = _param_pitch_setpoint.get() - pitch;
        float p_term = _param_p_gain.get() * error;
        float i_term = _integrator.update(error, dt);
        float d_term = _param_d_gain.get() * (-pitch_rate);  // derivative on measurement
        float torque_cmd = math::constrain(p_term + i_term + d_term,
                                           -_param_max_current.get(),
                                            _param_max_current.get());

        // 3. Publish current commands (Amps) to DRV8874 driver
        //    The inner current loop in the driver handles the actual PWM
        actuator_motors_s motors{};
        motors.timestamp = hrt_absolute_time();
        motors.control[0] = torque_cmd;   // left motor (Amps)
        motors.control[1] = torque_cmd;   // right motor (Amps, same for pure balance)
        _motors_pub.publish(motors);

        // 4. Publish status for logging
        balance_status_s status{};
        status.timestamp = hrt_absolute_time();
        status.pitch = pitch;
        status.pitch_rate = pitch_rate;
        status.torque_cmd = torque_cmd;
        status.error = error;
        _status_pub.publish(status);
    }
}
```

**Parameters (tunable via QGroundControl at runtime):**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `BAL_PITCH_P` | Float | 15.0 | Proportional gain (Amps per radian of pitch error) |
| `BAL_PITCH_I` | Float | 0.5 | Integral gain |
| `BAL_PITCH_D` | Float | 1.0 | Derivative gain |
| `BAL_PITCH_SP` | Float | 0.0 | Target pitch angle (rad, 0 = upright) |
| `BAL_MAX_I` | Float | 1.2 | Maximum current command (Amps, below 1.4A stall) |
| `BAL_LOOP_RATE` | Int | 500 | Control loop rate (Hz) |
| `BAL_I_MAX` | Float | 5.0 | Integrator anti-windup limit |

### 7.2 DRV8874 Motor Driver Module (Inner Current Loop)

**Location:** `src/drivers/actuators/drv8874/`

**Purpose:** Receives current commands from the balance controller, runs a fast inner PI loop to track the desired current using the Pololu CS pin for feedback, and drives the DRV8874 via PWM + GPIO.

**uORB subscriptions:**
| Topic | Data Used |
|-------|-----------|
| `actuator_motors` | Desired current per motor (Amps), channels 0 and 1 |

**uORB publications:**
| Topic | Data Published |
|-------|----------------|
| `motor_current` | Actual measured current per motor (for logging/diagnostics) |

**Inner current control loop (runs at PWM frequency, 10-20 kHz, from FlexPWM timer ISR):**

```cpp
// Called every PWM cycle (50 us at 20 kHz)
void DRV8874::current_control_isr()
{
    // 1. Read actual motor current from Pololu CS pin (ADC)
    //    Pololu: 2.49k RIPROPI, ~1.1 V/A
    //    R6 (1k) + C9 (10nF) filter: fc = 15.9 kHz
    float adc_voltage = adc_read_fast(_cs_adc_channel) * (3.3f / 4095.0f);
    float i_actual = adc_voltage / 1.133f;  // Convert to Amps

    // 2. Get desired current from balance controller (updated at 500-1000 Hz)
    float i_desired = _current_setpoint;  // Latched from uORB actuator_motors

    // 3. Determine direction
    bool forward = (i_desired >= 0.0f);
    float i_error = fabsf(i_desired) - i_actual;  // CS only reads magnitude

    // 4. Fast PI current controller
    float p_out = _kp_current * i_error;
    _i_integrator += _ki_current * i_error * _dt;
    _i_integrator = math::constrain(_i_integrator, 0.0f, _max_duty);
    float duty = math::constrain(p_out + _i_integrator, 0.0f, _max_duty);

    // 5. Apply to hardware
    set_direction_gpio(forward);   // PH/IN2 pin
    set_pwm_duty(duty);            // EN/IN1 pin

    // 6. Periodically publish actual current to uORB (at lower rate, e.g., 100 Hz)
    if (++_publish_counter >= _publish_divider) {
        _publish_counter = 0;
        motor_current_s msg{};
        msg.timestamp = hrt_absolute_time();
        msg.current[0] = i_actual;
        _current_pub.publish(msg);
    }
}
```

**Current PI parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `DRV_CUR_KP` | Float | 0.5 | Current loop proportional gain (duty per Amp error) |
| `DRV_CUR_KI` | Float | 50.0 | Current loop integral gain (higher = faster tracking) |
| `DRV_MAX_DUTY` | Float | 0.95 | Maximum PWM duty cycle (0-1) |
| `DRV_PWM_FREQ` | Int | 20000 | PWM frequency (Hz) |

**Hardware interface:**
```
i_desired > 0:  DIR GPIO = HIGH (forward), PWM duty from current PI output
i_desired < 0:  DIR GPIO = LOW  (reverse), PWM duty from current PI output
i_desired = 0:  PWM duty = 0 (brake via DRV8874 slow decay)
```

**Additional functions:**
- Set SLEEP GPIO HIGH on startup (after 1ms wake delay), LOW on shutdown
- Read FAULT GPIO -- if LOW, publish fault event and disable motor immediately
- ADC sampling synchronized to PWM midpoint (center of ON period) for cleanest current measurement
- PWM frequency: 20 kHz (configured via FlexPWM timer)

**CS pin hardware path:**
```
DRV8874 IPROPI → Pololu 2.49k to GND → CS pin → R6 (1k) + C9 (10nF) → Teensy ADC
                                         ~1.1 V/A    fc = 15.9 kHz
```

### 7.3 Wheel Encoder Module (Optional)

**Location:** `src/drivers/encoder/quadrature/`

**Purpose:** Read Hall encoder pulses from both motors, publish wheel odometry.

**uORB publications:**
| Topic | Data Published |
|-------|----------------|
| `wheel_encoders` | Counts per wheel, timestamps |
| `vehicle_odometry` | Distance traveled, speed (for EKF2 velocity aiding) |

**Encoder specs (Hiwonder JGA27-310R20):**
- 13 PPR magnetic Hall encoder
- 20:1 gear ratio
- Quadrature decoding: 13 x 20 x 4 = 1040 counts per output revolution
- Use GPIO interrupts on both channels for quadrature decoding
- The i.MX RT1062 has hardware quadrature decoder (ENC module) on some pins

---

## 8. uORB Data Flow Diagram

```
+------------------+       +------------------+       +------------------+
| ICM-20948 Driver |       | EKF2 Estimator   |       | Balance Control  |
| (existing PX4)   |       | (existing PX4)   |       | (CUSTOM: outer)  |
|                  |       |                  |       |  500-1000 Hz     |
| sensor_accel --->|------>| vehicle_attitude->|------>|                  |
| sensor_gyro  --->|------>|                  |       | actuator_motors->|
| sensor_mag   --->|------>|                  |       | (current cmds)   |
+------------------+       +------------------+       | balance_status ->|
                                                       +--------+---------+
                                                                |
                           +------------------+                 v
                           | PM02D INA226     |       +------------------+
                           | (existing PX4)   |       | DRV8874 Driver   |
                           |  10-50 Hz        |       | (CUSTOM: inner)  |
                           |  I2C bus 0       |       |  10-20 kHz       |
                           |                  |       |                  |
                           | battery_status ->|       | actuator_motors  |
                           +------------------+       |  + CS pin ADC    |
                                                       |  = Current PI    |
+------------------+       +------------------+       |  -> PWM + GPIO   |
| QGroundControl   |       | Logger (uLog)    |       | motor_current -> |
| (existing PX4)   |       | (existing PX4)   |       +------------------+
|                  |       |                  |                |
| parameter_update |       | <-- all topics   |                v
| MAVLink telem    |       |     to SD card   |       +------------------+
+------------------+       +------------------+       | Encoder Module   |
                                                       | (CUSTOM)         |
                                                       | wheel_encoders ->|
                                                       +------------------+
```

**Signal flow for torque control:**
```
EKF2            Balance PID          DRV8874 Current PI         Hardware
(500 Hz)        (500-1000 Hz)        (10-20 kHz)

pitch ──→ error ──→ I_desired ──→ I_error ──→ PWM duty ──→ DRV8874 EN pin
                                     ↑
                              I_actual (CS pin ADC)
```

**Existing PX4 modules** (no code changes needed):
- ICM-20948 SPI driver (accel + gyro + mag)
- EKF2 attitude estimator
- INA226 power monitor (PM02D on I2C, separate bus)
- MAVLink (telemetry over USB)
- Logger (uLog to SD card)
- Parameter system
- Commander (arming, state machine)

**Custom modules to write** (3 total):
1. `balance_control` -- Outer loop PID, outputs current commands (~200 lines)
2. `drv8874` -- Inner current PI loop + PWM/GPIO motor driver (~250 lines)
3. `quadrature_encoder` -- wheel odometry (~100 lines, optional)

---

## 9. Build and Flash Workflow

```bash
# 1. Clone PX4 with submodules
git clone --recursive https://github.com/PX4/PX4-Autopilot.git
cd PX4-Autopilot

# 2. Create custom board target (copy from tropic-community)
cp -r boards/nxp/tropic-community boards/custom/balancebot
# Edit configs as described in Sections 5.1-5.4

# 3. Add custom modules
mkdir -p src/modules/balance_control
mkdir -p src/drivers/actuators/drv8874
# Write module code as described in Section 6

# 4. Build
make custom_balancebot

# 5. Flash via Teensy bootloader
#    Press the button on the Teensy 4.1 to enter bootloader mode
make custom_balancebot upload_teensy
#    OR manually:
teensy-loader-cli --mcu=TEENSY41 -v -w build/custom_balancebot/custom_balancebot.hex

# 6. Connect QGroundControl
#    Teensy USB appears as MAVLink serial port
#    QGC auto-detects and connects
#    Tune BAL_PITCH_P, BAL_PITCH_I, BAL_PITCH_D in parameters tab
```

---

## 10. EKF2 Configuration for Ground Robot

EKF2 is designed for flying vehicles but works for ground robots with these adjustments:

| Parameter | Value | Reason |
|-----------|-------|--------|
| `EKF2_BARO_CTRL` | 0 | No barometer on board |
| `EKF2_GPS_CTRL` | 0 | No GPS on board |
| `EKF2_HGT_REF` | 0 | No height reference needed |
| `EKF2_MAG_TYPE` | 1 | Use magnetometer for heading |
| `EKF2_IMU_POS_X/Y/Z` | 0, 0, 0 | IMU at board center (above wheel axle) |
| `EKF2_GYR_NOISE` | 0.015 | ICM-20948 gyro noise density |
| `EKF2_ACC_NOISE` | 0.35 | ICM-20948 accel noise (may need increase for vibration) |

For the balance controller, the critical outputs are:
- **Pitch angle** from `vehicle_attitude.q` (quaternion -> Euler)
- **Pitch rate** from `vehicle_attitude.pitchspeed` (filtered gyro)

These are available at the EKF2 output rate (typically 250-500 Hz). For higher rates, the balance controller can also subscribe directly to `sensor_gyro` for raw 1 kHz gyro data and use EKF2 pitch only for drift correction.

---

## 11. What You Get vs. Arduino Approach

| Capability | Arduino/PlatformIO | PX4 on Same Hardware |
|------------|-------------------|---------------------|
| IMU driver | Write SPI code or use library | Existing, production-tested driver |
| Attitude estimation | Write Madgwick/Kalman filter | EKF2 (flight-grade, sensor-fused) |
| PID tuning | Recompile + reflash | QGroundControl parameter slider, live |
| Data logging | Serial.print() to terminal | Full uLog to SD, reviewable in FlightReview |
| Telemetry | Custom serial protocol | MAVLink + QGroundControl dashboard |
| Inter-module comms | Global variables / callbacks | uORB pub/sub (zero-copy, typed) |
| Task scheduling | loop() + millis() | NuttX RTOS work queues, priority-based |
| Fault handling | Custom code | Commander state machine (arming, failsafe) |
| Code structure | Single .ino file or ad-hoc | Module architecture, clean separation |
| Build system | PlatformIO / Arduino IDE | CMake + Make, reproducible builds |

**Tradeoff:** PX4 has a steeper initial setup (board target, NuttX config) but significantly more capability once running. The balance controller module itself is simple (~200 lines). Most of the effort is in the one-time board bring-up.

---

## 12. Implementation Phases

### Phase 1: Board Bring-Up (NuttX + PX4 Boot)
- [ ] Fork `nxp/tropic-community` to `custom/balancebot`
- [ ] Configure NuttX for correct SPI peripheral (LPSPI1 for Teensy SPI0)
- [ ] Map CS GPIO (Pin 10) and DRDY interrupt (Pin 17) in `board.h`
- [ ] Build and flash; verify PX4 boots and provides MAVLink shell over USB
- [ ] Run `icm20948 -s -b 1 start` from MAVLink shell; verify sensor data
- [ ] Confirm `sensor_accel` and `sensor_gyro` topics are publishing (`listener sensor_accel`)

### Phase 2: EKF2 Attitude
- [ ] Enable EKF2 in board config
- [ ] Set ground robot parameters (disable baro, GPS)
- [ ] Verify `vehicle_attitude` topic publishes valid pitch/roll/yaw
- [ ] Tilt the board by hand; confirm pitch angle tracks correctly in QGroundControl

### Phase 3: Motor Driver -- Open-Loop PWM First
- [ ] Write basic `drv8874` module (PWM + GPIO only, no current loop yet)
- [ ] Configure FlexPWM timers for 20 kHz on Pins 2 and 5
- [ ] Configure DIR GPIOs (Pins 3, 6) and SLEEP GPIOs (Pins 29, 30)
- [ ] Test: subscribe to `actuator_motors`, verify motors spin in correct direction
- [ ] Add FAULT GPIO monitoring
- [ ] Verify CS pin ADC reads valid current values (`listener motor_current`)

### Phase 4: Balance Controller -- Open-Loop (PWM) Balancing
- [ ] Write `balance_control` module with PID controller
- [ ] Subscribe to `vehicle_attitude`, publish to `actuator_motors` (as PWM duty, not current)
- [ ] Define tuning parameters (BAL_PITCH_P/I/D)
- [ ] Initial bench test: hold robot, verify motors respond correctly to tilt
- [ ] First free-standing balance test with PWM control
- [ ] Tune PID gains via QGroundControl parameter interface
- [ ] **Milestone: Robot balances with open-loop PWM control**

### Phase 5: Inner Current Loop -- Torque Control Upgrade
- [ ] Add current PI controller inside `drv8874` driver (10-20 kHz timer ISR)
- [ ] Configure ADC to sample CS pin synchronized to PWM midpoint
- [ ] Verify R6=1k + C9=10nF filter provides clean current signal (fc=15.9 kHz)
- [ ] Tune current loop: `DRV_CUR_KP`, `DRV_CUR_KI` with motor locked/stalled
- [ ] Switch `balance_control` output from PWM duty to current command (Amps)
- [ ] Test: robot should balance with more consistent torque across battery voltage range
- [ ] Compare uLog data: PWM control vs torque control (pitch variance, motor current)
- [ ] **Milestone: Robot balances with closed-loop torque control**

### Phase 6: Encoder + Odometry (Optional)
- [ ] Write quadrature encoder module for Pins 0/1 and 31/32
- [ ] Publish `wheel_encoders` topic
- [ ] Optionally feed wheel speed back to EKF2 for velocity estimation
- [ ] Use encoder data for position hold / drive-to-waypoint

### Phase 7: Polish
- [ ] Configure uLog to record balance_status, motor_current, actuator_motors, vehicle_attitude
- [ ] Review logs in PX4 Flight Review for tuning analysis
- [ ] Add safety: disable motors if pitch exceeds tilt limit (e.g., > 45 deg = fallen over)
- [ ] Add arming logic via Commander (button or QGC command to start balancing)
- [ ] Test battery voltage sweep: verify torque control maintains balance from 8.4V down to 6.0V

---

## 13. Key References

| Resource | URL / Location |
|----------|---------------|
| PX4 ICM-20948 driver source | `src/drivers/imu/invensense/icm20948/` in PX4-Autopilot repo |
| Tropic Community VMU board config | `boards/nxp/tropic-community/` in PX4-Autopilot repo |
| Tropic Community VMU hardware | https://github.com/NXP-Robotics/TropicCommunityVMU |
| PX4 NuttX porting guide | https://docs.px4.io/main/en/hardware/porting_guide_nuttx |
| PX4 board configuration guide | https://docs.px4.io/main/en/hardware/porting_guide_config |
| PX4 module template | https://docs.px4.io/main/en/modules/module_template |
| PX4 uORB messaging | https://docs.px4.io/main/en/middleware/uorb |
| PX4 actuator module guide | https://docs.px4.io/main/en/concept/pwm_limit |
| i.MX RT1062 reference manual | NXP IMXRT1060RM (from NXP website) |
| Teensy 4.1 pin mux table | `Docs/Teensy4.1 Pins Mux.pdf` (in project) |
| Schematic Design Guide | `Docs/Schematic_Design_Guide.md` (in project) |
| QGroundControl | https://docs.qgroundcontrol.com |
| PX4 Flight Review (log analysis) | https://review.px4.io |
