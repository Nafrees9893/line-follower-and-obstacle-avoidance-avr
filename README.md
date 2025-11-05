# Line-Following Robot (AVR) — with & without Obstacle Detection

This repo contains two versions of my line-following robot built **without a dev board** (bare-metal AVR):
1) **Line Follower (V1)** – simulated in Proteus and tested on real hardware  
2) **Line Follower + Obstacle Detection (V2)** – real hardware verified

https://github.com/Nafrees9893/line-follower-and-obstacle-avoidance-avr.git

---

## 🔥 Highlights
- **Bare-metal AVR (ATmega328P @ 16 MHz)** — no Arduino core
- **Three IR reflectance sensors** (L, C, R) with active-low logic
- **L298N dual H-bridge** motor driver
- **PWM speed control** (Timer1 & Timer2)
- **Status LEDs** for live sensor feedback
- **Two Versions**
  - **V1**: Line following (sim + hardware)
  - **V2**: Line following + obstacle detection (hardware)

---

## 🧠 How it Works (V1)
- IR sensors read the track (black = `0`, white = `1` via pull-ups).  
- Decision logic:
  - `Center=black`: go **straight** (FAST)
  - `Left=black`: **turn left** (slow left motor)
  - `Right=black`: **turn right** (slow right motor)
  - `Two sensors black`: **slow straight** (SLOW)
  - `None black`: **stop** (fail-safe)
- PWM sets motor speeds (`FAST=255`, `TURN≈190`, `SLOW≈175`).

**Core function (excerpt)**  
```c
void lineFollowLogic() {
    bool left   = !(PINC & (1<<LEFT_SENSOR));
    bool center = !(PINC & (1<<CENTER_SENSOR));
    bool right  = !(PINC & (1<<RIGHT_SENSOR));

    // LED feedback
    if (left)   PORTC |= (1<<GREEN_LED); else PORTC &= ~(1<<GREEN_LED);
    if (center) PORTC |= (1<<BLUE_LED);  else PORTC &= ~(1<<BLUE_LED);
    if (right)  PORTC |= (1<<RED_LED);   else PORTC &= ~(1<<RED_LED);

    uint8_t blackCount = left + center + right;

    if (blackCount == 1 && center)      moveForward(FAST_SPEED);
    else if (blackCount == 1 && left)   turnLeft(TURN_SPEED);
    else if (blackCount == 1 && right)  turnRight(TURN_SPEED);
    else if (blackCount == 2)           moveForward(SLOW_SPEED);
    else                                stopMotors();
}
