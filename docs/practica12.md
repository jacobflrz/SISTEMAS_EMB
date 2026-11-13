# Exam 1 - 20-LED Stacker Arcade Game via Direct SIO Bit-Packing and GPIO Interrupts

--- 

**Goal:** Build a complete 5-level by 4-column Stacker arcade game on the Raspberry Pi Pico 2 using a packed 32-bit integer board state, atomic single-cycle SIO register writes, and hardware falling-edge GPIO interrupts (`STOP` and `RESTART`) while keeping total worst-case current consumption well below `300 mA`.

**Prediction:** The active block will bounce horizontally across each 4-LED row without wrapping around. Pressing `STOP` will lock the overlapping LEDs via a bitwise `AND` against the level below, shrinking the block width and accelerating speed on each subsequent floor until reaching Level 5 (victory) or zero overlap (defeat), while `RESTART` will immediately reset the game from any state.

---

## Setup 

- Pin map:
  - **20-LED Matrix (5 Levels $\times$ 4 Columns):** Connected sequentially from `GP2` (Level 1, Column 0) through `GP21` (Level 5, Column 3), with each LED wired in series with its own current-limiting resistor (`220 Ω` or `330 Ω`) to common `GND`.
    - Level 1 (Bits `0`–`3` in `tablero`): `GP2`–`GP5`
    - Level 2 (Bits `4`–`7` in `tablero`): `GP6`–`GP9`
    - Level 3 (Bits `8`–`11` in `tablero`): `GP10`–`GP13`
    - Level 4 (Bits `12`–`15` in `tablero`): `GP14`–`GP17`
    - Level 5 (Bits `16`–`19` in `tablero`): `GP18`–`GP21`
  - **Control Pushbuttons (Active-Low with internal pull-ups):**
    - `STOP` Button (Lock active piece): `GP22`
    - `RESTART` Button (Immediate game reset): `GP26`

- Worst-Case Current Budget (`3.3 V` supply, all 20 LEDs asserted simultaneously):

| Component | Quantity | Resistor | Per-Pin Current Formula | Total Current |
|:---|:---:|:---:|:---|:---:|
| Green LEDs | `7` | `220 Ω` | `I = (3.3 V - 2.1 V) / 220 Ω = 5.45 mA` | `38.18 mA` |
| Yellow LEDs | `4` | `220 Ω` | `I = (3.3 V - 2.0 V) / 220 Ω = 5.91 mA` | `23.64 mA` |
| Red LED (Group A) | `1` | `220 Ω` | `I = (3.3 V - 2.0 V) / 220 Ω = 5.91 mA` | `5.91 mA` |
| Red LEDs (Group B) | `3` | `330 Ω` | `I = (3.3 V - 2.0 V) / 330 Ω = 3.94 mA` | `11.82 mA` |
| White LED | `1` | `220 Ω` | `I = (3.3 V - 3.0 V) / 220 Ω = 1.36 mA` | `1.36 mA` |
| Blue LEDs | `4` | `330 Ω` | `I = (3.3 V - 3.0 V) / 330 Ω = 0.91 mA` | `3.64 mA` |
| Active Button Pull-Up | `1` | `220 Ω` (ext) | `I = 3.3 V / 220 Ω = 15.00 mA` | `15.00 mA` |
| **Worst-Case Total** | **20 LEDs** | — | **Max single-pin LED load: `5.91 mA`** | **`99.55 mA` (< `300 mA`)** |

- Photo:
  ![Stacker Breadboard Setup](assets/exam1_stacker_setup.png)

- Non-default:
  - Internal pull resistors were explicitly disabled (`gpio_disable_pulls`) across all 20 LED output pins (`GP2`–`GP21`), while internal pull-ups (`gpio_pull_up`) were enabled on `GP22` and `GP26`.
  - All 20 LED states were updated atomically in a single clock cycle using `sio_hw->gpio_set` and `sio_hw->gpio_clr` with a 20-bit safety mask (`0xFFFFF << 2`).

---

## What we did 

1. Constructed the physical 5-row by 4-column LED display on the breadboard using 20 individual GPIOs (`GP2`–`GP21`), each paired with a dedicated current-limiting resistor, and verified that the worst-case current draw (`99.55 mA`) remained safely under the `300 mA` board limit.
2. Encoded the entire 20-LED game board inside a single 32-bit `volatile` integer (`tablero`), allocating 4 bits per floor (`fila * 4`) and masking with `0x0F` to isolate each row cleanly.
3. Implemented bidirectional horizontal scanning inside the main loop with boundary collision checks (`pieza & 0x08` on the left edge, `pieza & 0x01` on the right edge) and adaptive step delays (`150 - (fila * 20) ms`) that accelerate as the tower climbs.
4. Configured falling-edge interrupts (`GPIO_IRQ_EDGE_FALL`) on `GP22` (`STOP`) and `GP26` (`RESTART`) routed to `isr_botones()`.
5. Implemented bitwise alignment pruning (`uint8_t corte = pieza & abajo`) inside the `STOP` handler, using `__builtin_popcount(corte)` to shrink the active block width from 2 LEDs to 1 LED if only one column overlapped, or setting `fila = -1` (defeat) if `corte == 0`.
6. Added a 200 ms non-blocking timestamp debounce (`to_ms_since_boot(get_absolute_time())`) on `STOP` while giving `RESTART` immediate priority so a new game can be triggered from any state without lockouts.

---

## Evidence

### 20-LED Stacker Gameplay & Interrupt Demo

<video controls width="100%">
  <source src="../recursos/vids/Stacker_Game_demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

*Figure 1: Demonstration of the 5x4 LED Stacker game showing horizontal bouncing, block trimming on partial alignment, speed scaling per floor, victory flash, and instant restart.*

---

## Predicted vs measured

| Game Mechanic / Test | Input / Condition | Predicted Behavior | Observed Behavior | Status |
|:---|:---|:---|:---|:---:|
| Horizontal Bounce | Active piece reaches col 0 or 3 | Reverses direction without wrapping to adjacent row | Bounced cleanly within 4-bit row bounds | Verified |
| Exact Alignment (`STOP`) | `GP22` pressed directly over base | Locks full width, advances floor, speeds up (`-20 ms`) | Locked cleanly and accelerated on next level | Verified |
| Partial Overlap (`STOP`) | 1 of 2 LEDs overlaps lower row | Trims overhanging LED (`pieza & abajo`), shrinks width to `1` | Trimmed overhanging bit and continued with 1 LED | Verified |
| Zero Overlap (`STOP`) | Piece misses lower row (`corte == 0`) | Enters defeat state (`fila = -1`), freezes board | Halted game immediately until `RESTART` | Verified |
| Victory State | Level 5 locked with `corte != 0` | Sets `fila = 5`, illuminates all 20 LEDs (`0xFFFFF`) | All 20 LEDs lit solid upon completing floor 5 | Verified |
| Immediate Reset (`RESTART`) | `GP26` pressed during play/win/loss | Clears `tablero = 0`, resets to floor 0 with width 2 | Reset instantaneously without debounce lockout | Verified |

--- 

## What went wrong

- **Physical LED Array Wiring Complexity:** The main hardware challenge was assembling and routing the 20-LED matrix cleanly on the breadboard. We evaluated two topologies: driving each LED directly from its own dedicated GPIO versus building a multiplexed row/column matrix. Although direct GPIO wiring (`GP2`–`GP21`) required routing 20 individual current-limiting resistors and jumper leads across a dense breadboard layout, we chose direct pin mapping over multiplexing because it eliminated scanning flicker and allowed atomic single-cycle updates via `sio_hw->gpio_set` and `sio_hw->gpio_clr`.

- **Switch Bounce & Shared Debounce Lockouts:** During initial testing without software debouncing, mechanical contact bounce on the `STOP` button registered multiple falling edges from a single physical press, locking two or three consecutive floors at once. Adding a timestamp check (`ahora - t_stop < 200`) resolved multi-floor locking; however, when `STOP` and `RESTART` initially shared a single debounce timer, pressing `RESTART` immediately after `STOP` was ignored. Giving `RESTART` top priority in the ISR ahead of the `STOP` timestamp check eliminated post-reset lockouts.

---

## Code

```c
// Essential logic: atomic SIO refresh, bitwise block pruning, and horizontal bounce
static void pintar(void) {
    uint32_t bits = tablero;
    if (fila >= 0 && fila < 5) bits |= ((uint32_t)pieza << (fila * 4));
    if (fila == 5) bits = 0xFFFFF; // Victory: light all 20 LEDs

    sio_hw->gpio_set = (bits << LED_BASE) & MASCARA_TODOS_LEDS;
    sio_hw->gpio_clr = (~(bits << LED_BASE)) & MASCARA_TODOS_LEDS;
}

// Inside STOP ISR: bitwise intersection against the floor directly below
uint8_t abajo = (tablero >> ((fila - 1) * 4)) & 0x0F;
uint8_t corte = pieza & abajo;

if (!corte) { 
    fila = -1; // Defeat: zero overlap
    pintar(); 
    return; 
}

tablero |= ((uint32_t)corte << (fila * 4));
if (__builtin_popcount(corte) == 1) ancho = 1;

// Inside main loop: non-wrapping horizontal bounce within 4-bit row
if (dir == 1) {
    if (pieza & 0x08) { dir = -1; pieza >>= 1; }
    else pieza <<= 1;
} else {
    if (pieza & 0x01) { dir = 1;  pieza <<= 1; }
    else pieza >>= 1;
}
pieza &= 0x0F;
```
## Open Question

Since driving 20 LEDs directly consumes 20 GPIO pins on the Raspberry Pi Pico 2, how could we redesign the hardware using 5 row pins and 4 column pins (9 GPIOs total) with fast timer-driven persistence-of-vision multiplexing to achieve the same visual output while freeing up 11 pins?