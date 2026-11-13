# Session 9 - Bidirectional UART Communication Between Two Picos

--- 

**Goal:** Establish full-duplex asynchronous serial communication between two Raspberry Pi Pico 2 boards over `UART0` at 115200 baud, allowing pushbutton presses on either microcontroller to control corresponding status LEDs on the remote board.

**Prediction:** With crossed TX/RX lines and a shared common ground reference, pressing Button 1 or Button 2 on Pico 1 will transmit ASCII characters (`'0'`–`'3'`) that turn on LED 1 or LED 2 on Pico 2, and pressing the buttons on Pico 2 will produce the exact same response on Pico 1.

---

## Setup 

- Pin map (Identical on both Pico 1 and Pico 2):
  - UART Interface (`UART0` at 115200 baud):
    - `GP0` (Physical Pin 1, `UART0 TX`) on Pico 1 `->` `GP1` (Physical Pin 2, `UART0 RX`) on Pico 2.
    - `GP1` (Physical Pin 2, `UART0 RX`) on Pico 1 `->` `GP0` (Physical Pin 1, `UART0 TX`) on Pico 2.
  - Outputs: LED 1 = `GP12`, LED 2 = `GP13` (each wired in series with a 220\ &#x3A9; current-limiting resistor to GND).
  - Inputs: Button 1 = `GP14`, Button 2 = `GP15` (active-low pushbuttons using internal pull-up resistors).
  - Common Reference: Both Pico boards tied to the same ground (`GND`) rail on the breadboard.

- Photo:
  ![Setup Photo](assets/session9_setup.png)

- Non-default: Internal pull-up resistors (`gpio_pull_up`) were enabled on `GP14` and `GP15`, and both boards ran identical symmetrical firmware (`UART0` configured at 115200 baud).

---

## What we did 

1. Wired two LEDs (`GP12`, `GP13` with 220\ &#x3A9; resistors) and two active-low pushbuttons (`GP14`, `GP15`) on each Raspberry Pi Pico 2 board.
2. Cross-connected the `UART0` lines between both boards (`GP0` TX to `GP1` RX and vice versa) and joined their `GND` pins on a shared breadboard rail.
3. Initialized `UART0` at 115200 baud and flashed the exact same symmetrical C program onto both microcontrollers.
4. Configured the main loop to poll both buttons continuously and transmit status characters via `uart_putc()`: `'0'`/`'1'` for Button 1 and `'2'`/`'3'` for Button 2.
5. Implemented a non-blocking receiver check (`uart_is_readable()`) using `uart_getc()` to parse incoming characters and drive the local LEDs accordingly.
6. Tested bidirectional transmission from Pico 1 to Pico 2 and from Pico 2 to Pico 1.

---

## Evidence

### Bidirectional UART Demo

<video controls width="100%">
  <source src="../recursos/vids/UART_Two_Picos_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Video demonstration showing bidirectional UART control between two Raspberry Pi Pico 2 boards.*

---

## Predicted vs measured

| Sender Action | Transmitted Char | Receiver Pin | Predicted LED State | Observed Behavior | Status |
|:---|:---:|:---:|:---|:---|:---:|
| Button 1 Pressed (`GP14 = 0`) | `'0'` | `GP12` (LED 1) | ON (`1`) | Turned OFF (`0`) due to inverted logic in `if` block | Deviated |
| Button 1 Released (`GP14 = 1`) | `'1'` | `GP12` (LED 1) | OFF (`0`) | Stayed ON (`1`) while unpressed | Deviated |
| Button 2 Pressed (`GP15 = 0`) | `'2'` | `GP13` (LED 2) | ON (`1`) | Turned ON cleanly in both directions | Verified |
| Button 2 Released (`GP15 = 1`) | `'3'` | `GP13` (LED 2) | OFF (`0`) | Turned OFF cleanly in both directions | Verified |

--- 

## What went wrong

- **Common Ground (`GND`) Connection Issues:** Several initial communication failures occurred because we either forgot to tie the ground rails of both Pico boards together or had loose/intermittent ground jumper contacts on the breadboard. Without a solid shared voltage reference (`0 V`), UART logic thresholds floated unpredictably and prevented the receiving board from decoding frames reliably.

- **Inverted LED 1 Logic in Firmware:** In both directions, Button 2 properly turned on LED 2, whereas Button 1 behaved inversely (keeping LED 1 lit at rest and turning it off when pressed). Reviewing the receiver condition showed that `0` (Button 1 pressed) executed `gpio_put(LED1, 0)` while `1` (Button 1 released) executed `gpio_put(LED1, 1)`—the exact opposite of the active-high logic written for LED 2 (`2` `->` `1`, `3` `->` `0`).

---

## Code

```c
// Essential logic: continuous UART state transmission and character-mapped LED control
if (gpio_get(BTN1) == 0) {
    uart_putc(UART_ID, '0');
    sleep_ms(10);    
} else {
    uart_putc(UART_ID, '1');
    sleep_ms(10);    
}       

if (gpio_get(BTN2) == 0) {
    uart_putc(UART_ID, '2');
    sleep_ms(10);    
} else {
    uart_putc(UART_ID, '3');
    sleep_ms(10);    
}  

if (uart_is_readable(UART_ID)) {
    char c = uart_getc(UART_ID);
    if (c == '0') {
        gpio_put(LED1, 0);
    } else if (c == '1') {
        gpio_put(LED1, 1);
    } else if (c == '2') {
        gpio_put(LED2, 1);
    } else if (c == '3') {
        gpio_put(LED2, 0);
    }
}

```
## Open Question

Since both boards continuously stream button characters every 20 ms even when no button state changes, how could we send a single byte only on a button state transition (or pack both button states into one bitmask byte) to avoid filling up the receiver's UART FIFO buffer?