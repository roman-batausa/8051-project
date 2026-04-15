# 8051 LED Matrix Shooting Game

A fully playable shooting arcade game implemented in **8051 Assembly**, running on an 8x8 LED matrix display. The player controls a dual-gun turret at the bottom of the matrix, shoots down descending waves of enemies, and chooses between two difficulty levels — all with zero dependencies beyond the hardware itself.

---

## Demo

> *8×8 LED matrix driven via Port 0 (row data) and Port 1 (column select), with Port 3 buttons for player input.*

---

## Gameplay

- Enemies spawn at the **top of the matrix** and advance downward row by row.
- The player controls a **two-gun turret** that can slide left and right across the bottom two rows.
- Fire the **left gun** or **right gun** independently to shoot bullets upward.
- If enemies reach the **bottom row**, the game is over.
- Press any input button on the Game Over screen to **restart**.

---

## Features

| Feature                        | Details                                                                        |
|---                             |---                                                                             |
| **Dual Guns**                  | Left and right gun positions tracked independently                             |
| **Two Difficulty Modes**       | Selected before each game via animated menu                                    |
| **5 Progressive Enemy Speeds** | Enemies spawn at increasing rates as the wave count advances                   |
| **Two Shooting Mechanics**     | Level 1: destroy entire enemy column; Level 2: destroy one enemy dot per shot  |
| **Start Animation**            | Multi-frame intro sequence on game start                                       |
| **Game Over Animation**        | Animated heart/dissolve effect on loss                                         |
| **Restart**                    | Instant restart from the Game Over screen without power cycling                |

---

## Hardware

| Component                | Role                                      |
|---                       |---                                        |
| **8051 Microcontroller** | Main CPU                                  |
| **8×8 LED Matrix**       | Display output                            |
| **Port 0 (P0)**          | Row data (enemy/player/bullet pixel data) |
| **Port 1 (P1)**          | Column select (multiplexed scan)          |
| **Port 3 (P3.0–P3.3)**   | Player input buttons                      |

### Button Mapping

| Button | Action         |
|---     |---             |
| `P3.0` | Move Left      |
| `P3.1` | Move Right     |
| `P3.2` | Fire Right Gun |
| `P3.3` | Fire Left Gun  |

> During the **difficulty select screen**, `P3.0`/`P3.1` cycle the selection and `P3.2`/`P3.3` confirm it.

---

## Memory Map

The game state is stored entirely in the 8051's internal RAM using the bit-addressable and general-purpose register regions.

| Address   | Contents                                                 |
|---        |---                                                       |
| `20H–24H` | Enemy rows (top 5 rows of the matrix)                    |
| `25H`     | Enemy bottom row / collision boundary                    |
| `26H–27H` | Player turret shape (2 rows)                             |
| `2CH`     | Right gun bit position                                   |
| `2DH`     | Left gun bit position                                    |
| `2EH`     | Active column selector (rotating bit)                    |
| `2FH`     | Display refresh loop counter                             |
| `R0`      | Pointer into enemy/bullet rows (for shoot/destroy logic) |
| `R1`      | Pointer into display rows (for refresh scan)             |
| `R6`      | Frame counter (enemy spawn timer)                        |
| `R7`      | Enemy layer count                                        |

### Bit Flags (`20H` bit-addressable region)

| Bit         | Symbol           | Purpose                          |
|---          |---               |---                               |
| `0x40`      | LEFT_GUN         | Left gun fire pending            |
| `0x41`      | RIGHT_GUN        | Right gun fire pending           |
| `0x42`      | BULLET_TRAVELING | A bullet is currently in flight  |
| `0x43`      | MOVE_LEFT        | Left move action pending         |
| `0x44`      | MOVE_RIGHT       | Right move action pending        |
| `0x45`      | TOGGLE           | Debounce / one-shot input guard  |
| `0x46`      | LEVEL_ONE        | Level 1 mode active              |
| `0x47`      | LEVEL_TWO        | Level 2 mode active              |
| `0x48–0x4B` | SPEED_1–4        | Enemy wave speed stage flags     |

---

## Code Structure

```
184_PROJECT1.asm
│
├── START                    ; Init, bit clear, jump to difficulty select
├── START_GAME_ANIM          ; Reset state, play start FX, begin game loop
│
├── LED_MATRIX_LOOP          ; Main game loop
│   ├── REFRESH_DISP         ; Multiplex display + poll input
│   ├── CHECK_ACTION         ; Dispatch move/fire actions
│   ├── ACTION_ONCE          ; One-shot input debounce
│   ├── ADD_LAYER_1–5        ; Progressive enemy spawn stages
│   └── ADD_LAYER_FUNC       ; Shift enemy rows down, check collision
│
├── Player Movement
│   ├── MOVE_LEFT            ; Rotate player + gun positions left (RL A)
│   └── MOVE_RIGHT           ; Rotate player + gun positions right (RR A)
│
├── Shooting
│   ├── LEFT_GUN / RIGHT_GUN ; Load gun position into B
│   ├── SHOOT                ; AND bullet mask into current row, advance R0
│   ├── DESTROYED_LINE       ; XNOR + CPL — wipe entire enemy column (Level 1)
│   └── DESTROYED_ONE_DOT    ; XNOR + CPL — erase single enemy dot (Level 2)
│
├── Display
│   ├── GEN_DISP             ; Output P0 row → P1 column, rotate column selector
│   └── GEN_DISP_W_INPUT_CHECKER ; Same but also polls buttons (menus/game over)
│
├── Input
│   ├── CHECK_INPUT          ; Reads P3.x, sets bit flags
│   └── CHECK_RESTART        ; On game over screen, jump to START on input
│
├── Difficulty Select
│   ├── SELECT_DIFFICULTY_ONE  ; Animated "1" icon, Level 1 flag
│   └── SELECT_DIFFICULTY_TW0  ; Animated "2" icon, Level 2 flag
│
├── Animations
│   ├── START_FX             ; 15-frame startup animation sequence
│   └── LOSER_FX             ; Game over animation with heart dissolve
│
├── RESET_BITS               ; Clear all game state bit flags
└── DELAY                    ; Software delay loop (nested DJNZ)
```

---

## Game Loop Flow

```
Power On
   │
   ▼
SELECT_DIFFICULTY (animated menu, P3 input)
   │
   ▼
START_GAME_ANIM (play intro, reset state)
   │
   ▼
┌─ LED_MATRIX_LOOP ──────────────────────────────┐
│  1. REFRESH_DISP  → multiplex 8 columns + input │
│  2. CHECK_ACTION  → move / fire                 │
│  3. ACTION_ONCE   → one-shot debounce           │
│  4. Bullet/Destroy logic (if bullet in flight)  │
│  5. ADD_LAYER_x   → spawn enemy rows on timer   │
└────────────────────────────────────────────────┘
         │ Enemy reaches bottom row
         ▼
LOSER_FX (game over animation)
         │ P3 button pressed
         ▼
      START (restart)
```

---

## Building & Running

This project targets the **MCS-51 / 8051** instruction set.

### Simulate with Proteus / Keil µVision

1. Load the assembled `.hex` into an AT89C51 (or compatible) component.
2. Wire an 8×8 common-cathode LED matrix.
3. Connect four push buttons to **P3.0–P3.3** (active-low, pulled high).
4. Run the simulation.

> **Note:** A commented-out fast delay (`MOV 2BH, #1`) is included in the source for simulation speed. Uncomment it and comment out the hardware delay for simulation use.

---

## License

This project was developed as an academic exercise. Feel free to study, modify, and build upon it.
