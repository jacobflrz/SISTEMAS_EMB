# Session 5 - Emulating digital logic gates and implementing bidirectional LED shifting via SIO

--- 

**Goal:** Implement fundamental boolean logic gates (AND, OR, XOR) using digital inputs and low-level SIO register operations, and construct an interactive 4-LED bidirectional shift register controlled by two pushbuttons with edge detection and index wrapping.

**Prediction:** Direct SIO input register polling combined with bitwise boolean operators will emulate hardware logic gates with minimal propagation delay. In the moving LED system, rising-edge detection combined with modular arithmetic will prevent boundary overflow, ensuring the active bit shifts left or right without vanishing.

---

## Setup 

- Pin map: 
  - Inputs: Button A (Left) = `GP2`, Button B (Right) = `GP3` (configured as digital inputs with internal pull-down resistors enabled via `gpio_pull_down`).
  - Logic Gate Output: Status LED = `GP4`.
  - Moving LED Outputs: LED 1 = `GP4`, LED 2 = `GP5`, LED 3 = `GP6`, LED 4 = `GP7`.
  - Common ground tied between the Pico 2 GND rail and the breadboard bus.

- Photo:
  ![Setup Photo](assets/session5_setup.png)

- Non-default: Internal pull-downs were activated in software on `GP2` and `GP3` so that active inputs read as clean logic `1` upon closure without requiring discrete external pull-down resistors.

---

## What we did 

1. Initialized button pins as digital inputs and configured the LED pin masks as outputs via `sio_hw->gpio_oe_set`.
2. Extracted input bits from the 32-bit register `sio_hw->gpio_in` using bit-shifts and isolation masks (`(inputs >> PIN) & 1`).
3. Applied boolean bitwise operations (`&`, `|`, `^`) to emulate AND, OR, and XOR gates, driving the output LED using paired `gpio_set` and `gpio_clr` mask operations.
4. Expanded the hardware setup to 4 LEDs (`GP4` to `GP7`) and developed edge-detection logic (`a & (~prev_a)`) to capture button presses reliably.
5. Implemented index wrapping arithmetic (`(index + 1) % 4` for right shifts and `(index + 3) % 4` for left shifts) to bound the active LED position within the 4-pin range.
6. Compiled and flashed using the `Run` button in VS Code, confirming logic truth tables and interactive shift behavior.

---

## Evidence

### Logic Gates Demonstration (AND, OR, XOR)

<video controls width="100%">
  <source src="../recursos/vids/Logic_Gates_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Verification of boolean truth tables (AND, OR, XOR) using digital input switches and a single output LED.*

### Bidirectional Interactive Moving LED

<video controls width="100%">
  <source src="../recursos/vids/Moving_LED_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 2: Interactive LED position shifting left and right via button presses with cyclic boundary wrapping.*

---

## Predicted vs measured

| Function / Test | Input A (`GP2`) | Input B (`GP3`) | Predicted Output | Observed Behavior | Status |
|:---|:---:|:---:|:---:|:---|:---:|
| AND Gate | `1` | `1` | `1` (ON) | LED on only when both inputs are active | Verified |
| OR Gate | `1` / `0` | `0` / `1` | `1` (ON) | LED on when either or both inputs are active | Verified |
| XOR Gate | `1` | `1` | `0` (OFF) | LED turns off when both inputs match | Verified |
| Shift Right | Release | Press (Edge) | `index = (idx + 1) % 4` | Moves LED right; rolls from LED 4 to LED 1 | Verified |
| Shift Left | Press (Edge) | Release | `index = (idx + 3) % 4` | Moves LED left; rolls from LED 1 to LED 4 | Verified |

--- 

## What went wrong

- Utilizing a DIP switch module to input states for the logic gates complicated breadboard routing and mechanical connections. The close pin spacing made jumper connections prone to intermittent contacts, requiring careful rewiring and firm seating to ensure stable readings.

- Developing the logic for the moving LED without having the active bit disappear proved challenging. Standard bitwise shifts (`<<=` or `>>=`) easily pushed the active bit beyond the 4-pin boundary, turning off all LEDs permanently. Structuring the position as an index with modular boundary wrapping (`+1 % 4` and `+3 % 4`) resolved the overflow issue cleanly.

---

## Code

### 1. SIO Logic Gate Evaluation (AND / OR / XOR)

```
// Sampled from the core loop: input bit extraction and gate evaluation
int inputs = sio_hw->gpio_in;

int a = (inputs >> PIN_BTN_A) & 1;
int b = (inputs >> PIN_BTN_B) & 1;

// Selected operator: '&' for AND, '|' for OR, '^' for XOR
int result = a ^ b; 

int led_bit = result << PIN_LED;

sio_hw->gpio_set = led_bit;
sio_hw->gpio_clr = (~led_bit) & MASK_OUT;

```

### 2. LED Shifting

```
int a_pressed = a & (~prev_a); // Rising edge: Left button
int b_pressed = b & (~prev_b); // Rising edge: Right button

if (b_pressed & 1) {
    index = (index + 1) % 4; // Shift right with wrap-around
}

if (a_pressed & 1) {
    index = (index + 3) % 4; // Shift left (-1 mod 4) with wrap-around
}

prev_a = a;
prev_b = b;

sio_hw->gpio_clr = MASK_OUT;
sio_hw->gpio_set = (1 << (PIN_LED_1 + index));

```

## Open question

- Currently, two separate GPIOs are allocated exclusively for directional input sensing. Could an analog input (ADC) with a resistor divider ladder or a single bidirectional pin detect both directions, freeing up GPIO pins without complicating the edge-detection logic?