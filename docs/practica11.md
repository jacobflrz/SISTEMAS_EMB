# Session 11 - I2C Bus Communication: Real-Time Clock (DS1307) and Accelerometer (MPU6050)

--- 

**Goal:** Utilize the `I2C0` bus of the Raspberry Pi Pico in the Wokwi simulator at `400 kHz` to communicate with two peripheral slave devices sharing the same serial lines: reading timekeeping registers (`hours`, `minutes`, `seconds`) from a **DS1307 Real-Time Clock (RTC)** and 3-axis acceleration (`X`, `Y`, `Z`) from an **MPU6050 IMU**.

**Prediction:** With both slave devices addressed independently on the shared `I2C0` bus, the DS1307 time readout will increment cleanly by one second per sampling cycle, while the MPU6050 at rest will report approximately `0 g` on the horizontal `X` and `Y` axes and `1 g` (`16384` raw counts at `±2 g` full-scale range) along the vertical `Z` axis due to gravity.

---

## Setup 

- Pin map (Wokwi Simulation using independent `I2C0` and `I2C1` buses at `400 kHz`):
  - **DS1307 RTC (`I2C0`, Address `0x68`):** `GP4` (`SDA`), `GP5` (`SCL`), powered from `VBUS` (`5 V`).
  - **MPU6050 IMU (`I2C1`, Address `0x68`):** `GP2` (`SDA`), `GP3` (`SCL`), powered from `3V3` (`3.3 V`).
  - **Common Ground:** `GND` pins of the Pi Pico, DS1307, and MPU6050 tied together.

- Photo:
  ![Wokwi Circuit Setup](assets/session11_wokwi_setup.png)

- Non-default:
  - Allocated two separate hardware I2C controllers (`i2c0` and `i2c1`) to avoid a `0x68` slave address conflict between both modules.
  - Internal pull-up resistors (`gpio_pull_up`) were enabled on all four serial lines (`GP2`, `GP3`, `GP4`, `GP5`), and both buses were initialized at Fast Mode (`400 kHz`).
---

## What we did 

## What we did 

1. Wired the DS1307 RTC to `I2C0` (`GP4`/`GP5`) and the MPU6050 IMU to `I2C1` (`GP2`/`GP3`) in Wokwi, isolating both devices onto separate channels so their identical `0x68` default addresses would not collide.
2. Initialized both `I2C0` and `I2C1` at `400 kHz` with internal pull-ups enabled and woke the MPU6050 from sleep mode by writing `0x00` to its power management register (`0x6B`).
3. **DS1307 Time Acquisition (`I2C0`):** Sent a single-byte write pointing to register `0x00` using a repeated start (`nostop = true`), then read `3` sequential bytes (`seconds`, `minutes`, `hours`).
4. Masked control bits (`CH` clock-halt bit 7 in seconds and 12/24-hour mode bits in hours) and converted the Binary-Coded Decimal (BCD) nibbles into standard decimal integers (`((bcd >> 4) * 10) + (bcd & 0x0F)`).
5. **MPU6050 Acceleration Acquisition (`I2C1`):** Pointed to the starting data register `0x3B` (`ACCEL_XOUT_H`) and burst-read `6` consecutive bytes, combining high and low bytes into signed 16-bit integers (`int16_t`) for `X`, `Y`, and `Z`.
6. Printed both the formatted clock (`HH:MM:SS`) and raw/scaled acceleration values to the Serial Monitor every `1000 ms`.

---

## Evidence

### Wokwi I2C Simulation & Serial Telemetry

<video controls width="100%">
  <source src="../recursos/vids/I2C_RTC_MPU6050_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Wokwi simulation recording showing simultaneous 1-second RTC increments and 3-axis accelerometer readings over I2C0.*

---

## Predicted vs measured

| Device / Parameter | I2C Address & Register | Predicted Value | Measured Raw Output | Scaled / Decoded Output | Status |
|:---|:---:|:---:|:---:|:---|:---:|
| **DS1307 Time** (`HH:MM:SS`) | `0x68` / Reg `0x00` (`3 B`) | Advances `+1 s` per cycle | BCD bytes (`0x00`–`0x59`) | Increments `1 s` per reading cleanly | Verified |
| **MPU6050 Accel X** (At rest) | `0x68` / Reg `0x3B` (`2 B`) | `0.00 g` (`0` counts) | `0` | `0 / 16384 = 0.00 g` | Verified |
| **MPU6050 Accel Y** (At rest) | `0x68` / Reg `0x3D` (`2 B`) | `0.00 g` (`0` counts) | `0` | `0 / 16384 = 0.00 g` | Verified |
| **MPU6050 Accel Z** (At rest) | `0x68` / Reg `0x3F` (`2 B`) | `+1.00 g` (`16384` counts) | `16384` | `16384 / 16384 = +1.00 g` | Verified |

--- 

## What went wrong

- Nothing went wrong during this practice; the simulated `I2C0` bus handled repeated-start register addressing cleanly, BCD bit-masking decoded the clock registers accurately, and the MPU6050 reported exact resting gravity counts.

---

## Code

```c
// Essential logic: BCD-to-decimal conversion and DS1307 bit-masking
uint8_t bcd_a_decimal(uint8_t bcd) {
    uint8_t mascara_unidades = (1 << 4) - 1; 
    return ((bcd >> 4) * 10) + (bcd & mascara_unidades);
}

// Clear bit 7 (Clock Halt flag) on seconds register
rtc_data[0] &= ~(1 << 7); 
uint8_t segundos = bcd_a_decimal(rtc_data[0]); 
uint8_t minutos  = bcd_a_decimal(rtc_data[1]);

// Mask upper bits to isolate 24-hour format digits
rtc_data[2] &= ((1 << 6) - 1); 
uint8_t horas = bcd_a_decimal(rtc_data[2]);

```

## Open question

Currently, the Pico polls both `I2C0` and `I2C1` sequentially inside the main loop every second. How could we use the MPU6050's hardware `INT` pin to trigger an interrupt only when new acceleration data is ready, avoiding unnecessary bus transactions?