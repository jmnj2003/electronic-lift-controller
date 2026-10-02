# Electronic Lift Controller System

A gate-level digital logic design of an **8-floor hotel lift controller**, built and simulated in [DEEDS](https://www.digitalelectronicsdeeds.com/). A 3-bit saturating up/down counter made from D flip-flops tracks the lift's floor. Around it, combinational logic handles password and room-card access, door safety, a master enable that gates the clock, status indicators, and an emergency stop.

> Group project for **SMJE1113 Digital Electronics**, Bachelor in Artificial Intelligence, Universiti Teknologi Malaysia (Semester 2, 2025/2026).

▶️ **Demo video:** https://youtu.be/7mtNhtzHTdw

<p align="center">
  <img src="assets/full_circuit.png" alt="Full lift controller circuit in DEEDS: password and card decoders, door control, master enable, and the 3-bit counter" width="780">
</p>

---

## How it works

A hotel guest enters a password, taps a room card, selects a floor, closes the door and presses UP or DOWN. The lift counts floor by floor until it reaches the selected floor. Then it stops by itself, opens the door and sounds the alarm.

<p align="center">
  <img src="assets/block_diagram.png" alt="Block diagram: user inputs, password check, card verification, door control, master enable, clock enabler, counter, comparator, stop/open/alarm/display" width="560">
</p>

The lift only moves when **all four** of these conditions are true at once:

```
Master_Enable = ENABLE · Password_OK · Card_OK · Door_Closed
```

`Master_Enable` gates the clock that drives the counter's flip-flops. If any condition fails (wrong password digit, door open, already at the destination), clock pulses stop reaching the counter and the lift holds its floor.

## Design

### 1. Floor counter: 3-bit saturating up/down counter

- Three positive-edge-triggered **D flip-flops**, with outputs `Q2 Q1 Q0` for floors `000` to `111` (0–7).
- Direction input `U`: `1` counts up, `0` counts down.
- **Saturating:** it holds at floor 7 when told to go up and at floor 0 when told to go down. It never wraps around.

<p align="center">
  <img src="assets/state_diagram.png" alt="State diagram with 8 states, U=1 counts up, U=0 counts down, hold at 111 and 000" width="360">
</p>

Next-state logic, worked out from the state table → Karnaugh maps → minimised sum-of-products:

```
D2 = Q2·Q0 + Q2·Q1 + Q2·U + Q1·Q0·U
D1 = Q2·Q1'·Q0'·U' + Q2·Q1·U + Q1'·Q0·U + Q1·Q0'·U + Q1·Q0·U'
D0 = Q2·Q1·U + Q2·Q0' + Q1·Q0' + Q0'·U
```

All 16 (state, direction) transitions have been checked against the state table, including both saturation holds.

### 2. Equality comparator and clock enabler

- Three **XNOR** gates compare the current floor (`Q2 Q1 Q0`) with the selected floor (`S2 S1 S0`), and an **AND** gate combines them: `EQUAL = 1` when the lift has arrived.
- `ENABLE = EQUAL'` blocks the clock on arrival, so the counter stops at the right floor automatically.

### 3. Direction control latch

Real lift buttons are pressed once, not held down. A D flip-flop is used as a latch through its **asynchronous** `PR'` / `CL'` inputs:

- UP sets `Q = 1` and DOWN clears `Q = 0` immediately, without waiting for a clock edge.
- With no button pressed, the flip-flop holds the direction for the whole trip.
- `D` is tied to 0 and `Ck` to 1, so the synchronous path is disabled.

<p align="center"><img src="assets/direction_control.png" alt="Direction control D flip-flop latch using PR' and CL'" width="520"></p>

### 4. Access control: password and room card

- **Password:** three hex digits, each decoded by a **4-to-16 decoder**. The output line for the correct digit from each decoder feeds a 3-input AND, giving `Password_OK`.
- **Room card:** the same decoder approach, giving `Card_OK`. Using the same MSI component keeps both security checks consistent.
- The demo circuit uses password `123` and card ID `ABC`.

### 5. Door control

A single **NOR** gate opens the door either manually or automatically on arrival:

```
Door_Closed = (Door_Switch + EQUAL)'
```

| Door_Switch | EQUAL | Door_Closed |
|:-:|:-:|:-:|
| 0 | 0 | 1 (closed) |
| 1 | 0 | 0 (open) |
| 0 | 1 | 0 (open, arrived) |
| 1 | 1 | 0 (open) |

### 6. Emergency mode

```
FINAL_CLOCK = CLOCK · EMERGENCY'
```

EMERGENCY stops the clock instead of driving `CLR`/`PRE`, so the lift **freezes at its current floor** and is not reset to floor 0. It also forces ALARM and DOOR OPEN on.

### 7. Status indicators and display

<p align="center"><img src="assets/status_indicators.png" alt="Status indicators: access granted/denied, door open/closed, arrived, alarm, lift moving, and 7-segment displays" width="620"></p>

| Indicator | Logic |
|---|---|
| Current floor (7-seg) | `Q2 Q1 Q0` |
| Selected floor (7-seg) | `S2 S1 S0` |
| ACCESS GRANTED | `Password_OK · Card_OK` |
| ACCESS DENIED | `(Password_OK · Card_OK)'` (NAND) |
| ARRIVED | `EQUAL` |
| LIFT MOVING | `Final_Clock · Master_Enable` |
| DOOR CLOSED | `(Door_Switch + EQUAL)' · EMERGENCY'` |
| DOOR OPEN | `(Door_Switch + EQUAL) + EMERGENCY` |
| ALARM | `EQUAL + EMERGENCY` |

## Testing

18 test cases were run in DEEDS covering the displays, access granted/denied, door states, lift moving/stopped, arrival, alarm, emergency stop and recovery after an emergency. **All 18 passed.** Changing any single master-enable condition to an invalid value (for example one wrong password digit, or the door left open) froze the counter as expected.

## Running the simulation

1. Install **DEEDS** (Digital Electronics Education and Design Suite) and open `circuit/lift_controller.pbs` in the **d-DcS** digital circuit simulator.
2. Start the simulation, then:
   1. Enter the password `1 2 3` and the card ID `A B C`.
   2. Pulse **CLR** once so the counter starts at floor 0.
   3. Press **UP** or **DOWN**, then pulse **CLK** to settle the direction latch.
   4. Set the destination floor with switches **S2 S1 S0**, and make sure the door switch is `0` (closed).
   5. Clock the circuit. The lift counts toward the destination, stops, and turns on ARRIVED, ALARM and DOOR OPEN.
   6. Press **EMERGENCY** at any time to stop the lift where it is.

> Flip-flops start in an undefined state when a DEEDS simulation starts, so always do the CLR + one clock pulse step first.

## Project structure

```
├── circuit/
│   └── lift_controller.pbs     # DEEDS d-DcS circuit file
├── docs/
│   └── presentation.pdf        # project slides
└── assets/                     # diagrams and circuit screenshots used in this README
```

## My contribution

I designed and implemented the **direction control latch**, **password authentication**, **hotel card verification**, **door control** and **master enable logic**: the access and safety layer that decides when the lift is allowed to move. I also wrote the system requirements.

## Team

Mun Weng Yann · Nik Muhammad Faris bin Nik Zaki · Bong Xin Ting · **Jessie Moh Ngiik Jun**

Lecturer: Ts. Dr. Sahnius binti Usman, Section EB10

## Possible improvements

- Automatic door timer instead of a manual door switch
- Queue of multiple floor requests (memory)
- Overload detection
- A real buzzer output and a richer display

## References

- *Digital Electronics Deeds*. https://www.digitalelectronicsdeeds.com/
- Rahman, M. et al. (2017). *A Simulation Study of an Elevator Control System using Digital Logic.* International Journal of Engineering Trends and Technology, 52(3).
