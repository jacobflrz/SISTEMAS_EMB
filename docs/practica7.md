# Session 8 - Fairground-Style LED Roulette using GPIO Interrupts

--- 

**Goal:** Build an interactive fairground-style LED roulette on the Raspberry Pi Pico 2 using falling-edge GPIO interrupts (`GPIO_IRQ_EDGE_FALL`), direct SIO register manipulation, and `volatile` state variables to dynamically adjust scanning speed and capture a target win condition.

**Prediction:** A single active bit will scan back and forth across a 5-LED bus at a step interval governed by `tiempo_espera`. Pressing the speed-control buttons will trigger hardware interrupts that modify the step delay in real time, while pressing the stop button at the exact moment the center LED (`GP12`) is asserted will halt the sequence and trigger a victory blink routine.

---

## Setup 

- Pin map: 
  - Output LED Bus (5-LED array): LED 1 = `GP10`, LED 2 = `GP11`, LED 3 (Target Center) = `GP12`, LED 4 = `GP13`, LED 5 = `GP14` (cathodes tied to GND through current-limiting resistors).
  - Input Pushbuttons (Active-Low with internal pull-ups):
    - Button 1 (Slow down): `GP15`
    - Button 2 (Stop / Win trigger): `GP16`
    - Button 3 (Speed up): `GP17`
  - Common ground tied between the Pico 2 GND rail and the breadboard bus.

- Photo:
  ![Setup Photo](assets/session7_setup.png)

- Non-default: Internal pull-up resistors were enabled via `gpio_pull_up()` on `GP15`, `GP16`, and `GP17`, allowing the buttons to connect directly to GND without external pull-up networks.

---

## What we did 

1. Initialized the 5-LED bus (`GP10`–`GP14`) using `gpio_init_mask(PINS_MASK)` and enabled outputs through `sio_hw->gpio_oe_set`.
2. Configured three pushbuttons (`GP15`, `GP16`, `GP17`) as digital inputs with internal pull-ups enabled and attached falling-edge interrupts (`GPIO_IRQ_EDGE_FALL`) routed to a shared callback (`rutina_boton`).
3. Implemented `volatile` global variables (`corriendo` and `tiempo_espera`) so changes executed inside the ISR propagate immediately to the main scanning loop without compiler optimization caching.
4. Built a non-blocking interruptible delay routine (`pausa()`) that slices the wait time into 1 ms increments and exits immediately if `corriendo` becomes `false`.
5. Verified the target win condition by reading the live output register (`sio_hw->gpio_out & (1u << PINS[2])`) inside the ISR when `GP16` is pressed, triggering a rapid 150 ms celebration blink on the center LED upon success.
6. Tested speed increments/decrements and calibrated step bounds to ensure playable transitions.

---

## Evidence

### Roulette Gameplay & Speed Adjustment Demo

<video controls width="100%">
  <source src="../recursos/vids/LED_Roulette_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Video demonstration showing real-time speed adjustments via GP15/GP17 and winning target capture on the center LED (GP12) via GP16.*

---

## Predicted vs measured

| Input / Action | Pin | Trigger Event | Predicted Behavior | Observed Behavior | Status |
|:---|:---:|:---:|:---|:---|:---:|
| Slow Down Button | `GP15` | Falling Edge | Increases delay (`+400 ms`, max `2600 ms`) | Visibly slows LED scan rate | Verified |
| Speed Up Button | `GP17` | Falling Edge | Decreases delay (`-400 ms`, min `400 ms`) | Visibly accelerates LED scan rate | Verified |
| Stop (Off-Center) | `GP16` | Falling Edge | `sio_hw->gpio_out` check fails; keeps running | Sequence continues uninterrupted | Verified |
| Stop (Center LED) | `GP16` | Falling Edge | `GP12` is active; sets `corriendo = false` | Stops sequence and blinks center LED | Verified |

--- 

## What went wrong

- Adjusting the LED scanning speed presented noticeable calibration issues during initial testing. Using overly small delay increments made speed changes barely perceptible to the player, whereas removing strict upper and lower bounds either caused the delay variable to underflow (wrapping around to a massive unsigned integer value and freezing the LED) or made the sequence move too fast to track visually. Adding explicit boundary clamps (`> 400` ms and `< 2600` ms) with 400 ms steps alongside mechanical button bounce stabilization made the speed control responsive and playable.

---

## Code

```c
// Essential logic: shared ISR for speed control and center-pin win validation
volatile bool corriendo = true;
volatile uint32_t tiempo_espera = 1000;

static void rutina_boton(uint gpio, uint32_t events) {
    if (gpio == boton2) {
        // Win condition: verify center LED (GP12) is currently asserted in SIO output
        if (sio_hw->gpio_out & (1u << PINS[2])) {
            corriendo = false;
        }
    } 
    else if (gpio == boton1) {
        if (tiempo_espera < 2600) {
            tiempo_espera += 400; // Slower
        }
    } 
    else if (gpio == boton3) {
        if (tiempo_espera > 400) {
            tiempo_espera -= 400; // Faster
        }
    }
    gpio_acknowledge_irq(gpio, events);
}

// Interruptible delay allowing immediate stop response mid-step
void pausa(uint32_t ms) {
    for (uint32_t i = 0; i < ms; i++) {
        if (!corriendo) return;
        sleep_ms(1);
    }
}
```
## Open question

How can we implement a simple software debounce inside the button ISR using `time_us_32()` so that one press only changes the speed by a single step?