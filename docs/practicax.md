# Session 7 - Hardware Repeating Timers and Pulse Event Counting

--- 

**Goal:** Configure a hardware repeating timer (`repeating_timer_t`) on the Raspberry Pi Pico 2 to generate a periodic square-wave time-base signal (`2000 ms` toggle interval) while simultaneously counting asynchronous falling-edge pulses from a pushbutton and reporting the accumulated count over USB Serial at each timer tick.

**Prediction:** The repeating timer interrupt will toggle `GP0` every `2000 ms`, producing a steady `4000 ms` (`0.25 Hz`) square wave that defines a fixed sampling window. During each `2 s` window, pressing the pushbutton on `GP1` will generate discrete falling-edge pulses counted by the main loop, and every timer callback will print the total count before resetting the accumulator to zero.

---

## Setup 

- Pin map:
  - Square-Wave Time-Base Output: `GP0` (`TEST_PIN`, monitored with an oscilloscope / logic analyzer or status LED).
  - Asynchronous Pulse Input: `GP1` (`BUTTON_PIN`, active-low pushbutton configured with internal pull-up resistor to `3.3 V` and switch to `GND`).
  - Serial Telemetry: USB CDC Serial Monitor at 115200 baud (`pico_enable_stdio_usb` enabled in `CMakeLists.txt`).

- Photo:
  ![Setup Photo](assets/session12_setup.png)

- Non-default:
  - Passed a negative interval (`-TOGGLE_MS = -2000`) to `add_repeating_timer_ms()` so the hardware alarm schedules callbacks strictly from the start of each execution (`2000 ms` period) rather than adding callback execution latency between triggers.
  - Enabled the internal pull-up resistor (`gpio_pull_up`) on `GP1` to keep the input idle HIGH (`1`).

---

## What we did 

1. Initialized `GP0` as a digital output (`TEST_PIN`) to visualize the periodic timer window and configured `GP1` (`BUTTON_PIN`) as an active-low input with `gpio_pull_up()`.
2. Registered a repeating hardware timer callback (`timer_callback`) using `add_repeating_timer_ms(-2000, timer_callback, NULL, &timer)`.
3. Inside `timer_callback()`, toggled `GP0` (`gpio_put(TEST_PIN, !gpio_get(TEST_PIN))`) to produce a square wave marking each `2000 ms` sampling interval, printed the accumulated `count` over USB Serial, and reset `count = 0`.
4. Implemented falling-edge detection (`prev && !curr`) inside the non-blocking `while (true)` loop to increment the `volatile uint32_t count` variable every time the button was pressed.
5. Observed both signals (the periodic timer square wave on `GP0` and the asynchronous button pulses on `GP1`) alongside the Serial Monitor output every 2 seconds.

---

## Evidence

### Timer Square Wave & Button Pulse Counting Demo

<video controls width="100%">
  <source src="../recursos/vids/Timers_Pulse_Counter_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Video demonstration showing the 2000 ms square-wave interval toggle on GP0 and real-time pulse counts printed per window on the Serial Monitor.*

---

## Predicted vs measured

| Signal / Parameter | Pin / Channel | Predicted Behavior | Observed Behavior | Status |
|:---|:---:|:---|:---|:---:|
| Timer Interval Square Wave | `GP0` (`TEST_PIN`) | Toggles state every `2000 ms` (`0.25 Hz` full cycle) | Clean `2000 ms` high / `2000 ms` low square wave | Verified |
| Button Pulse Input | `GP1` (`BUTTON_PIN`) | Generates active-low pulse (`1 -> 0`) on press | Registered falling-edge transitions into `count` | Verified |
| Windowed Serial Report | USB CDC (`stdio`) | Prints accumulated `count` every `2000 ms` and resets | Reported exact pulse count per 2 s window and zeroed | Verified |

--- 

## What went wrong

- Nothing went wrong during this practice; the repeating timer maintained an exact `2000 ms` toggle period, the square-wave intervals matched the expected timing windows, and the pulse counter reported button presses accurately without errors.

---

## Code

```c
// Essential logic: 2000 ms repeating timer callback and falling-edge pulse counter
static volatile uint32_t count = 0;

static bool timer_callback(repeating_timer_t *rt) {
    gpio_put(TEST_PIN, !gpio_get(TEST_PIN));
    printf("Pulsaciones: %lu\n", (unsigned long)count);
    count = 0;
    return true; // Keep repeating timer active
}

// Inside main loop: non-blocking falling-edge detection
bool prev = true;
while (true) {
    bool curr = gpio_get(BUTTON_PIN);
    if (prev && !curr) count++;
    prev = curr;
    tight_loop_contents();
}
```