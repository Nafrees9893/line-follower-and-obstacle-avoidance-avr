#define F_CPU 16000000UL
#include <avr/io.h>
#include <util/delay.h>
#include <stdbool.h>

// Sensor and LED definitions
#define LEFT_SENSOR    PC0
#define CENTER_SENSOR  PC1
#define RIGHT_SENSOR   PC2

#define GREEN_LED      PC4
#define BLUE_LED       PC5
#define RED_LED        PC3  // Changed RED_LED from PD5 to PC3

// Motor control definitions
#define ENA_PIN PD3
#define IN1_PIN PD4
#define IN2_PIN PB0
#define ENB_PIN PB1
#define IN3_PIN PB2
#define IN4_PIN PB3

// Motor speeds
#define FAST_SPEED 255
#define TURN_SPEED 190
#define SLOW_SPEED 175

void initIO() {
	// Motor control pins
	DDRD |= (1<<ENA_PIN) | (1<<IN1_PIN);
	DDRB |= (1<<IN2_PIN) | (1<<ENB_PIN) | (1<<IN3_PIN) | (1<<IN4_PIN);

	// LED outputs
	DDRC |= (1<<GREEN_LED) | (1<<BLUE_LED) | (1<<RED_LED);

	// Sensor inputs with pull-ups
	DDRC &= ~((1<<LEFT_SENSOR) | (1<<CENTER_SENSOR) | (1<<RIGHT_SENSOR));
	PORTC |= (1<<LEFT_SENSOR) | (1<<CENTER_SENSOR) | (1<<RIGHT_SENSOR);
}

void initPWM() {
	// PWM on ENA - Timer2
	TCCR2A |= (1<<COM2B1) | (1<<WGM20);
	TCCR2B |= (1<<CS21); // Prescaler 8

	// PWM on ENB - Timer1
	TCCR1A |= (1<<COM1A1) | (1<<WGM10);
	TCCR1B |= (1<<CS11); // Prescaler 8
}

void setSpeed(uint8_t left, uint8_t right) {
	OCR2B = left;
	OCR1A = right;
}

void moveForward(uint8_t speed) {
	PORTD |= (1<<IN1_PIN);
	PORTB &= ~(1<<IN2_PIN);
	PORTB |= (1<<IN3_PIN);
	PORTB &= ~(1<<IN4_PIN);
	setSpeed(speed, speed);
}

void turnLeft(uint8_t speed) {
	PORTD |= (1<<IN1_PIN);
	PORTB &= ~(1<<IN2_PIN);
	PORTB |= (1<<IN3_PIN);
	PORTB &= ~(1<<IN4_PIN);
	setSpeed(speed / 2, speed); // Left slower
}

void turnRight(uint8_t speed) {
	PORTD |= (1<<IN1_PIN);
	PORTB &= ~(1<<IN2_PIN);
	PORTB |= (1<<IN3_PIN);
	PORTB &= ~(1<<IN4_PIN);
	setSpeed(speed, speed / 2); // Right slower
}

void stopMotors() {
	PORTD &= ~(1<<IN1_PIN);
	PORTB &= ~(1<<IN2_PIN);
	PORTB &= ~(1<<IN3_PIN);
	PORTB &= ~(1<<IN4_PIN);
	setSpeed(0, 0);
}

void lineFollowLogic() {
	// Sensor logic (active low: black = 0)
	bool left   = !(PINC & (1<<LEFT_SENSOR));
	bool center = !(PINC & (1<<CENTER_SENSOR));
	bool right  = !(PINC & (1<<RIGHT_SENSOR));

	// LED feedback
	if (left)   PORTC |= (1<<GREEN_LED); else PORTC &= ~(1<<GREEN_LED);
	if (center) PORTC |= (1<<BLUE_LED);  else PORTC &= ~(1<<BLUE_LED);
	if (right)  PORTC |= (1<<RED_LED);   else PORTC &= ~(1<<RED_LED);

	uint8_t blackCount = left + center + right;

	if (blackCount == 1 && center) {
		moveForward(FAST_SPEED);
	}
	else if (blackCount == 1 && left) {
		turnLeft(TURN_SPEED);
	}
	else if (blackCount == 1 && right) {
		turnRight(TURN_SPEED);
	}
	else if (blackCount == 2) {
		moveForward(SLOW_SPEED);
	}
	else {
		stopMotors();
	}
}

int main(void) {
	initIO();
	initPWM();

	while (1) {
		lineFollowLogic();
	}
}
