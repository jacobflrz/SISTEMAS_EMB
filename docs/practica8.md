# Session 8 - Building a Monophonic PWM Piano with Variable Frequency and Volume

--- 

**Goal:** Build a monophonic 7-note piano on the Raspberry Pi Pico 2 using hardware Pulse Width Modulation (PWM) to control pitch via clock dividers and perceived volume via duty cycle adjustment.

**Prediction:** Each musical note will sound through the buzzer exclusively while its assigned pushbutton is held down, silencing immediately upon release. Adjusting the duty cycle via two dedicated volume pushbuttons will modulate acoustic amplitude without altering the pitch frequency.

---

## Setup 

- Pin map: 
  - Output: Passive/Active Buzzer = `GP2` (configured as PWM output slice/channel with `TOP = 16383`).
  - Note Inputs (7 natural notes, active-low with internal pull-ups):
    - Note C = `GP10`, Note D = `GP11`, Note E = `GP12`, Note F = `GP13`, Note G = `GP14`, Note A = `GP15`, Note B = `GP16`.
  - Volume Inputs (active-low with internal pull-ups):
    - Volume Down = `GP20`
    - Volume Up = `GP21`
  - Common ground shared between the Pico 2 GND rail, buzzer negative terminal, and pushbutton return rails.

- Photo:
  ![Setup Photo](assets/session8_setup.png)

- Non-default: Internal pull-ups (`gpio_pull_up`) were enabled across all nine input GPIOs (`GP10`–`GP16`, `GP20`, `GP21`), and the PWM wrap ceiling (`TOP`) was set to `16383` ($2^{14} - 1$) to extend low-frequency reach within the RP2350's 8-bit integer clock divider constraint ($\text{div}_{\max} = 255$).

---

## What we did 

1. Wired the buzzer to `GP2`, the seven natural-note pushbuttons to `GP10`–`GP16`, and the two volume control buttons to `GP20` and `GP21`.
2. Extracted the seven natural frequencies (`C`, `D`, `E`, `F`, `G`, `A`, `B`) from a single column of the standard musical pitch chart—selecting **Octave 1** (`32.70 Hz` to `61.74 Hz`) rather than **Octave 5** (`523.25 Hz` to `987.77 Hz`)[cite: 1].
3. Configured `GP2` for PWM output with `TOP = 16383` and computed the required clock divider dynamically from the system clock ($f_{\text{clk}} = 150\text{ MHz}$) using:
   $$f_{\text{PWM}} = \frac{f_{\text{clk}}}{\text{div} \cdot (\text{TOP} + 1)} \implies \text{div} = \frac{f_{\text{clk}}}{f_{\text{PWM}} \cdot (\text{TOP} + 1)}$$
4. Clamped `div` strictly to the hardware bounds $[1.0, 255.0]$ and updated both `pwm_set_clkdiv()` and `pwm_set_chan_level()` whenever an active key was detected.
5. Implemented a monophonic priority scan across the seven note buttons so that only the first pressed key sounds, setting `duty_level = 0` to silence the buzzer whenever all keys are released.
6. Compiled and flashed the firmware using the `Run` button in VS Code and tested note playback and volume stepping.

---

## Evidence

### Note Frequency Chart Reference

![Musical Note Frequency Chart](../recursos/imgs/note_frequency_chart.jpg)

*Figure 1: Standard equal-tempered note frequency reference table across Octaves 0–8 used to extract the single-octave natural scale.*

### PWM Piano Operation Demo

<video controls width="100%">
  <source src="../recursos/vids/PWM_Piano_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 2: Demonstration of button-triggered PWM tones and duty-cycle volume modulation on GP2.*

---

## Predicted vs measured

| Natural Note | Pin | Selected Octave 1 ($f_{\text{target}}$) | Target Octave 5 ($f_{\text{ideal}}$) | Calculated `div` (Octave 1) | Clamped Hardware `div` | Observed Output | Status |
|:---:|:---:|:---:|:---:|:---:|:---:|:---|:---:|
| **C** | `GP10` | `32.70 Hz`| `523.25 Hz`| `279.96` | `255.0` (Saturated) | Clamped at ~35.86 Hz / Inaudible | Deviated |
| **D** | `GP11` | `36.71 Hz`| `587.33 Hz`| `249.38` | `249.38` | Very low / below buzzer resonance | Deviated |
| **E** | `GP12` | `41.20 Hz`| `659.26 Hz`| `222.20` | `222.20` | Very low / faint buzz | Deviated |
| **F** | `GP13` | `43.65 Hz`| `698.46 Hz`| `209.73` | `209.73` | Very low / faint buzz | Deviated |
| **G** | `GP14` | `49.00 Hz`| `783.99 Hz`| `186.83` | `186.83` | Very low / faint buzz | Deviated |
| **A** | `GP15` | `55.00 Hz`| `880.00 Hz`| `166.45` | `166.45` | Audible low fundamental tone | Verified |
| **B** | `GP16` | `61.74 Hz`| `987.77 Hz`| `148.28` | `148.28` | Audible low fundamental tone | Verified |

--- 

## What went wrong

- **Incorrect Octave Selection (Octave 1 vs. Octave 5):** When isolating a single octave column from the frequency reference chart, we extracted the seven natural notes from **Octave 1** (`32.70 Hz` to `61.74 Hz`) instead of **Octave 5** (`523.25 Hz` to `987.77 Hz`). Because the RP2350 system clock runs at `150 MHz` and `TOP = 16383`, the lowest attainable frequency before hitting the hardware clock divider ceiling (`div_max = 255`) is: f_min = 150,000,000 / (255 * 16384) = 35.90 Hz
Consequently, note C1 (32.70 Hz) saturated the divider at 255.0, and notes D1 through G1 (36.71 Hz to 49.00 Hz) fell well below the acoustic resonant band of standard piezo buzzers (typically 1 kHz to 4 kHz). Only the two highest notes in that octave (A1 at 55.00 Hz and B1 at 61.74 Hz) produced clearly perceptible sound, though at a very low rumble that made pitch accuracy difficult to verify by ear.

---

## Code

```c
float f_clk = (float)clock_get_hz(clk_sys);

float div = f_clk / (freq * (TOP + 1));
// Restricción de hardware (1.0 <= div <= 255.9375)
if (div > 255.0f) div = 255.0f;
if (div < 1.0f) div = 1.0f;

// Aplicar frecuencia recalculando el divisor
pwm_set_clkdiv(slice, div);

// Aplicar volumen actual por duty cycle
pwm_set_chan_level(slice, chan, duty_level);
note_pressed = true;
break;
    }
}
// Si se sueltan todas las teclas, silenciar (duty cycle a 0)
if (!note_pressed) {
    pwm_set_chan_level(slice, chan, 0);
}

```

## Open question

Why does a buzzer reach its loudest volume at a 50% PWM duty cycle instead of 100%, and how would adding a second PWM slice allow playing two notes at the same time (polyphony)?