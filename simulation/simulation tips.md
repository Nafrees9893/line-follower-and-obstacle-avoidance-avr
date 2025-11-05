Simulation (V1)

Tool: Proteus

Open simulation/proteus/line_follower.pdsprj.

MCU → set Clock = 16 MHz.

Compile firmware (see Build) and load the main.hex.

Ensure IR modules are active-low; adjust thresholds if using analog variants.

Run — verify sensor LEDs and motor outputs.

⚙️ Build & Flash (avr-gcc + avrdude)

Requirements

avr-gcc, avr-libc, avrdude

Programmer: USBasp / Arduino as ISP / etc.