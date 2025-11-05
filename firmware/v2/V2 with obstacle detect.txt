#ifndef F_CPU
#define F_CPU 16000000UL
#endif

#include <avr/io.h>
#include <util/delay.h>
#include <avr/interrupt.h>
#include <stdint.h>

// ===== Motor pins (direction) =====
#define MOTOR_LEFT_FORWARD     PB2
#define MOTOR_LEFT_BACKWARD    PB1
#define MOTOR_RIGHT_FORWARD    PB3
#define MOTOR_RIGHT_BACKWARD   PB4

// ===== L298 enable pins (PWM) =====
#define ENA PD6   // OC0A -> Left motor enable
#define ENB PD5   // OC0B -> Right motor enable

// ===== IR sensor pins =====
#define LEFT_SENSOR   PC0
#define CENTER_SENSOR PC1
#define RIGHT_SENSOR  PC2

// ===== Ultrasonic sensor pins =====
#define TRIG_PIN  PD0
#define ECHO_PIN  PD1

// ===== LED pins =====
#define LED1 PD4  // Mode 1 indicator
#define LED2 PD7  // Mode 2 indicator
#define LED_OBSTACLE PD3  // Obstacle detection indicator

// ===== Mode control =====
volatile uint8_t mode = 0;  // 0=OFF, 1=LF only, 2=LF + Ultrasonic
volatile uint8_t button_irq_flag = 0; // set by INT0 on button press

// ===== Helper macros =====
#define SET(PORT, PIN)    (PORT |= (1 << PIN))
#define CLEAR(PORT, PIN)  (PORT &= ~(1 << PIN))

// ===== PWM / Speed control =====
static inline void pwm_init(void){
	DDRD |= (1 << ENA) | (1 << ENB);
	TCCR0A = (1 << WGM01) | (1 << WGM00) | (1 << COM0A1) | (1 << COM0B1);
	TCCR0B = (1 << CS01); // prescaler=8 (~7.8 kHz)
	OCR0A = 0;
	OCR0B = 0;
}
static inline void set_speed(uint8_t left, uint8_t right){
	OCR0A = left;
	OCR0B = right;
}

// Suggested speed levels
#define SPEED_SLOW      150
#define SPEED_TURN      170
#define SPEED_STRAIGHT  180

// ===== Motor Functions =====
void move_forward(void){
	SET(PORTB, MOTOR_LEFT_FORWARD);
	CLEAR(PORTB, MOTOR_LEFT_BACKWARD);
	SET(PORTB, MOTOR_RIGHT_FORWARD);
	CLEAR(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(SPEED_STRAIGHT, SPEED_STRAIGHT);
}
void pivot_left(void){
	SET(PORTB, MOTOR_LEFT_FORWARD);
	CLEAR(PORTB, MOTOR_LEFT_BACKWARD);
	CLEAR(PORTB, MOTOR_RIGHT_FORWARD);
	SET(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(SPEED_TURN, SPEED_TURN);
}
void pivot_right(void){
	CLEAR(PORTB, MOTOR_LEFT_FORWARD);
	SET(PORTB, MOTOR_LEFT_BACKWARD);
	SET(PORTB, MOTOR_RIGHT_FORWARD);
	CLEAR(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(SPEED_TURN, SPEED_TURN);
}
void turn_left(void){
	CLEAR(PORTB, MOTOR_LEFT_FORWARD);
	CLEAR(PORTB, MOTOR_LEFT_BACKWARD);
	SET(PORTB, MOTOR_RIGHT_FORWARD);
	CLEAR(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(0, SPEED_TURN);
}
void turn_right(void){
	SET(PORTB, MOTOR_LEFT_FORWARD);
	CLEAR(PORTB, MOTOR_LEFT_BACKWARD);
	CLEAR(PORTB, MOTOR_RIGHT_FORWARD);
	CLEAR(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(SPEED_TURN, 0);
}
void stop(void){
	CLEAR(PORTB, MOTOR_LEFT_FORWARD);
	CLEAR(PORTB, MOTOR_LEFT_BACKWARD);
	CLEAR(PORTB, MOTOR_RIGHT_FORWARD);
	CLEAR(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(0, 0);
}
// NEW: brief reverse helper
void move_backward(void){
	CLEAR(PORTB, MOTOR_LEFT_FORWARD);
	SET(PORTB, MOTOR_LEFT_BACKWARD);
	CLEAR(PORTB, MOTOR_RIGHT_FORWARD);
	SET(PORTB, MOTOR_RIGHT_BACKWARD);
	set_speed(SPEED_SLOW, SPEED_SLOW);
}

// ===== Ultrasonic Functions =====
void ultrasonic_init(void){
	DDRD |= (1 << TRIG_PIN);
	DDRD &= ~(1 << ECHO_PIN);
}

uint16_t get_distance(void){
	CLEAR(PORTD, TRIG_PIN);
	_delay_us(2);
	SET(PORTD, TRIG_PIN);
	_delay_us(10);
	CLEAR(PORTD, TRIG_PIN);

	uint16_t timeout = 0;
	while (!(PIND & (1 << ECHO_PIN)) && timeout < 60000) timeout++;
	if (timeout >= 60000) return 0; // no echo

	uint32_t pulse = 0;
	while (PIND & (1 << ECHO_PIN)) {
		_delay_us(1);
		pulse++;
		if (pulse > 40000) break;
	}
	uint16_t distance = (uint16_t)((pulse * 343UL) / 20000UL); // cm
	return distance;
}

// ===== Interrupts =====
// Single push-button on PD2 (INT0)
ISR(INT0_vect){
	button_irq_flag = 1;
}

static inline void handle_button_press(void){
	if(!button_irq_flag) return;

	// --- Debounce: wait 30ms, verify still pressed (active LOW) ---
	_delay_ms(30);
	if(!(PIND & (1 << PD2))){
		// Toggle sequence: 0 -> 1 -> 2 -> 1 -> 2 ...
		if(mode == 0)         mode = 1;
		else if(mode == 1)    mode = 2;
		else                  mode = 1;

		// Wait for release (non-blocking enough; short loop)
		while(!(PIND & (1 << PD2))) _delay_ms(5);
	}
	button_irq_flag = 0;
}

// ===== Main =====
int main(void){
	// Motor direction pins OUTPUT
	DDRB |= (1 << MOTOR_LEFT_FORWARD) | (1 << MOTOR_LEFT_BACKWARD) |
	(1 << MOTOR_RIGHT_FORWARD) | (1 << MOTOR_RIGHT_BACKWARD);

	// IR sensor pins INPUT with pull-ups
	DDRC &= ~((1 << LEFT_SENSOR) | (1 << CENTER_SENSOR) | (1 << RIGHT_SENSOR));
	PORTC |= (1 << LEFT_SENSOR) | (1 << CENTER_SENSOR) | (1 << RIGHT_SENSOR);

	// Ultrasonic & PWM init
	ultrasonic_init();
	pwm_init();

	// LED pins OUTPUT
	DDRD |= (1 << LED1) | (1 << LED2) | (1 << LED_OBSTACLE);

	// Button PD2 as input with pull-up
	DDRD &= ~(1 << PD2);
	PORTD |= (1 << PD2);

	// External interrupt on INT0 (falling edge)
	EICRA |= (1 << ISC01);     // ISC01=1, ISC00=0 -> falling edge
	EIMSK |= (1 << INT0);
	sei();

	while(1){
		// Process button presses (debounced)
		handle_button_press();

		// Update LEDs based on mode
		if(mode == 1){ SET(PORTD, LED1); CLEAR(PORTD, LED2); CLEAR(PORTD, LED_OBSTACLE); }
		else if(mode == 2){ SET(PORTD, LED2); CLEAR(PORTD, LED1); }
		else { CLEAR(PORTD, LED1); CLEAR(PORTD, LED2); CLEAR(PORTD, LED_OBSTACLE); }

		if(mode == 0){  // No mode selected
			stop();
			continue;
		}

		// Ultrasonic check only in mode 2: stop and blink LED if obstacle within 10 cm
		if(mode == 2){
			uint16_t distance = get_distance();
			if(distance > 0 && distance <= 10){
				stop();
				// Blink obstacle LED (100ms ON, 100ms OFF) twice
				for(uint8_t i = 0; i < 2; i++){
					SET(PORTD, LED_OBSTACLE);
					_delay_ms(100);
					CLEAR(PORTD, LED_OBSTACLE);
					_delay_ms(100);
				}
				continue;
				} else {
				CLEAR(PORTD, LED_OBSTACLE); // Ensure LED is off when no obstacle
			}
		}

		// Read IR sensors
		uint8_t sensors = PINC;
		// Inputs are pulled-up: 1=WHITE, 0=BLACK; keep same mapping
		uint8_t left   = !((sensors & (1 << LEFT_SENSOR))   ? 0 : 1);
		uint8_t center = !((sensors & (1 << CENTER_SENSOR)) ? 0 : 1);
		uint8_t right  = !((sensors & (1 << RIGHT_SENSOR))  ? 0 : 1);

		// NEW: if all three see BLACK, back up briefly (20 ms), then continue with normal logic
		if(!left && !center && !right){
			move_backward();
			_delay_ms(50);
			pivot_right();
			_delay_ms(20);
		}

		// ===== Line following logic (unchanged) =====
		if(left && center && right)        move_forward();
		else if(center && left)            pivot_right();
		else if(center && right)           pivot_left();
		else if(center)                    move_forward();
		else if(left)                      turn_left();
		else if(right)                     turn_right();
		else                               stop();

		_delay_ms(10);
	}
	return 0;
}