Hardware

MCU: ATmega328P (16 MHz crystal, 22 pF caps)

Motor Driver: L298N

Motors: 2× DC gear motors + wheels

Sensors: 3× IR line sensors (digital)

Power: 2S/3s Li-ion or 6–9 V DC (separate logic/motor rails recommended)

Extras: 3× LEDs (left/center/right), on/off switch, mode switch (optional)

Optional for V2 (Obstacle)

Ultrasonic: HC-SR04 (or similar)

Servo (if you sweep the sensor)

Mode switch: toggle Line-Follow vs Avoid


🔌 Wiring (summary)

PC0/PC1/PC2 → IR sensors (L/C/R) with internal pull-ups

PD3 (ENA) PWM (Timer2B) → L298N ENA

PD4 (IN1), PB0 (IN2) → Left motor direction

PB1 (ENB) PWM (Timer1A) → L298N ENB

PB2 (IN3), PB3 (IN4) → Right motor direction

PC3/PC4/PC5 → LEDs (R/G/B)

16 MHz crystal on XTAL1/2, VCC/AVCC decoupled (100 nF near pins)