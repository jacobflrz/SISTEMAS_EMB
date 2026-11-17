# Session 11 - Analog-to-Digital Conversion (ADC): Potentiometer, Temperature, and Brightness

--- 

**Goal:** Utilize the 12-bit Analog-to-Digital Converter (ADC) on the Raspberry Pi Pico 2 to sample analog sensors on `GP26` (`ADC0`) and convert raw digital counts (`0` to `4095`) into meaningful physical units: potentiometer angular displacement (`0°`–`270°`), ambient temperature (`°C`), and relative brightness (`%`).

**Prediction:** Sweeping the potentiometer across its mechanical travel will produce linear raw values from `0` to `4095` (`0 V` to `3.3 V`), mapping directly to `0°`–`270°`. Warming the LM35 temperature sensor will linearly increase its output voltage (`10 mV/°C`), while covering or illuminating the photoresistor (LDR) will shift the ADC reading between minimum and maximum brightness percentages.

---

## Setup 

- Pin map:
  - Analog Input: `GP26` (`ADC0`, Physical Pin 31) connected sequentially to each sensor output:
    - Exercise 1: Rotary Potentiometer (wiper to `GP26`, outer pins to `3.3 V` and `GND`).
    - Exercise 2: LM35 Analog Temperature Sensor (`V_out` to `GP26`, `VCC` to `3.3 V` / `VBUS`, `GND` to common ground).
    - Exercise 3: Photoresistor (LDR) voltage divider node connected to `GP26`.
  - Serial Telemetry: USB CDC Serial Monitor at 115200 baud (`pico_enable_stdio_usb` enabled in `CMakeLists.txt`).

- Photo:
  ![Setup Photo](assets/session10_setup.png)

- Non-default:
  - Enabled USB stdio output in `CMakeLists.txt` to stream real-time formatted telemetry over the Serial Monitor.
  - Applied a 16-sample circular moving average filter (`N_SAMPLES = 16`) on the potentiometer and brightness exercises to attenuate high-frequency ADC quantization noise.

---

## What we did 

1. Initialized the RP2350 ADC peripheral via `adc_init()`, configured `GP26` for analog input (`adc_gpio_init(26)`), and selected channel `0` (`adc_select_input(0)`).
2. **Exercise 1 (Potentiometer Angle):** Sampled raw 12-bit counts (`0`–`4095`) and converted them into voltage and shaft rotation (`0°`–`270°`) using floating-point scaling (`voltage = raw * 3.3 / 4095`, `degrees = (raw / 4095.0) * 270.0`), printing telemetry every `100 ms`.
3. **Exercise 2 (Temperature Sensor):** Read the LM35 analog output every `1000 ms`, converting raw counts to voltage (`raw * 3.3 / 4096`) and dividing by the sensor's linear scale factor (`0.01 V/°C`) to display temperature in Celsius.
4. **Exercise 3 (Brightness Percentage):** Sampled the light sensor using a 16-sample moving average, mapping the full `0`–`4095` range into a `0%`–`100%` brightness scale printed every `200 ms` under dark (covered) and bright (direct light) test conditions.
5. Verified the Serial Monitor readouts and adjusted integer/floating-point arithmetic to prevent truncation errors.

---

## Evidence

### Potentiometer & Brightness ADC Telemetry Demo

<video controls width="100%">
  <source src="../recursos/vids/ADC_Sensors_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Video demonstration showing real-time Serial Monitor conversion of raw ADC counts into angle degrees and brightness percentage.*

---

## Predicted vs measured

| Exercise / Sensor | Input Condition | Predicted Raw (`0`–`4095`) | Predicted Unit Conversion | Observed Behavior | Status |
|:---|:---|:---:|:---|:---|:---:|
| 1. Potentiometer | Full CCW (`0°`) $\rightarrow$ Full CW (`270°`) | `0` $\rightarrow$ `4095` | `0.0 V`–`3.3 V` (`0°`–`270°`) | Smooth linear sweep after fixing integer division | Verified |
| 2. Temperature (LM35) | Room temp (~`24 °C`) & finger heat | ~`298` (`0.24 V`) | ~`24.0 °C` (`10 mV/°C`) | Erratic/shifted readings due to sensor overvoltage | Deviated |
| 3. Brightness (LDR) | Covered (Dark) vs. Direct Light | Low (~`200`) $\rightarrow$ High (~`3900`) | `~5 %` $\rightarrow$ `~95 %` | Responsive percentage tracking across light levels | Verified |
| 4. Sound Sensor (Extra) | Ambient / Clap acoustic input | Variable AC waveform | Acoustic amplitude / dB | Omitted (microphone module unavailable) | N/A |

--- 

## What went wrong

- **Missing Sound Sensor Hardware (Extra Practice):** An additional exercise required measuring ambient sound levels, but we could not complete it because we did not have a microphone module with an integrated pre-amplifier available during the session.

- **LM35 Temperature Sensor Overvoltage:** During Exercise 2, the LM35 temperature sensor was inadvertently overvolted (in addition to prior hardware stress on the component), which damaged its internal calibration and produced erratic, inaccurate temperature readings on the Serial Monitor.

- **Integer Truncation in Moving Average Calculation:** In the potentiometer exercise, using an earlier version of the 16-sample circular moving average (`uint16_t average = sum / N_SAMPLES`) alongside integer division in the scaling formulas caused decimal truncation (`raw / 4095` evaluating to `0` for any value below `4095`). Bypassing the integer average truncation and ensuring explicit floating-point division (`3.3f / 4095.0f`) restored accurate voltage and angle calculations.

---

## Code

```c
// Essential logic: 12-bit ADC sampling and linear physical unit conversions
adc_init();
adc_gpio_init(26);
adc_select_input(0);

// 1. Potentiometer: raw count to voltage and angular degrees (0 - 270 deg)
uint16_t raw = adc_read();
float voltage = (raw * 3.3f) / 4095.0f;
float degrees = (raw / 4095.0f) * 270.0f;

// 2. LM35 Temperature: 10 mV per degree Celsius
float temp_voltage = (raw * 3.3f) / 4096.0f;
float temperature_c = temp_voltage / 0.01f;

// 3. Brightness: linear percentage across 12-bit dynamic range
float brightness_percent = (raw / 4095.0f) * 100.0f;
```
## Open Question

Since the RP2350 ADC uses a 3.3 V reference voltage (AREF) that can fluctuate slightly due to USB power rail noise, how can we use an external precision voltage reference or oversampling techniques to improve the accuracy of small-signal analog sensors like the LM35?