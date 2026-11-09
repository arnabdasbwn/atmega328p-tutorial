---
title: "Using The Timer HAL"
description: "Configure CTC, fixed and variable-period PWM, input capture and stop operations through the course interface."
id: timer-hal
prerequisites: [timer0, timer1, timer2, interrupts]
experiments: [07-timer0-ctc, 08-timer1-ctc, 09-timer2-ctc, 11-pwm-led, 12-servo, 38-input-capture]
validation: "API limits, timer ownership and stop behaviour reviewed against the HAL header/implementation; representative consumers build with XC8 and AVR GCC"
---

# Using the Timer HAL

![Timer compare-match path from the system clock through the prescaler and counter to the event flag](../assets/diagrams/timer-compare.svg)

## Timer Configuration Interface

The HAL declares timer functions in [`avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h) and implements them in [`timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c). `avr_timer_ctc()` configures one timer for a compare interval. `avr_timer_pwm8()` sets fixed 8-bit fast PWM. `avr_timer1_pwm()` configures variable-period PWM on Timer1 D9. `avr_timer1_capture()` prepares input capture on D8. `avr_timer_stop()` stops the selected timer clock and disables its interrupts.

Each function reports invalid timer or unsupported configuration values instead of silently choosing another mode. The exact supported prescalers, TOP and duty ranges are in the public header. The HAL owns the entire selected timer when configuring it; it disables that timer's interrupts and does not enable global interrupts.

## Ownership and Safe Shutdown

The output-compare pins depend on the timer: Timer0 uses D6/D5, Timer1 uses D9/D10 and Timer2 uses D11/D3. Those pins may also be used for GPIO or another peripheral function. Check the board pin map before enabling a timer output.

`avr_timer_stop()` is not an actuator-safe shutdown. The timer stops, but its compare configuration and GPIO latch remain as they were. Before stopping a motor or other load, explicitly set the driver to its inactive state and disconnect the PWM output as required.

## Choose a Function for the Task

- Use CTC for a fixed interval or event rate.
- Use 8-bit PWM for a fixed timer period and variable duty.
- Use Timer1 variable-period PWM when a longer frame or a servo-style pulse is required.
- Use input capture to timestamp an external edge.

Read the example's `lab.json` and the HAL header before combining features. A timer is a shared hardware resource; two independently working features may still conflict when linked together.

## Exercise with Answer

Does `avr_timer_stop(AVR_TIMER1)` guarantee that a motor connected to OC1A becomes electrically inactive?

**Answer:** No. The API stops the timer clock but leaves output and pin state unchanged. The application must explicitly put the motor driver into a safe state.

## Code Walkthrough


**Example code on GitHub:**
- [Timer0 polling timebase (`07-timer0-ctc`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/07-timer0-ctc/main.c)
- [Timer1 one-second compare (`08-timer1-ctc`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/08-timer1-ctc/main.c)
- [Timer2 independent clock divider (`09-timer2-ctc`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/09-timer2-ctc/main.c)
- [PWM LED brightness (`11-pwm-led`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/11-pwm-led/main.c)
- [SG90 servo pulse generation (`12-servo`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/12-servo/main.c)
- [Timer1 hardware edge timestamp (`38-input-capture`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/38-input-capture/main.c)

### Timer0 polling timebase
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `uint16_t count = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `(void)avr_gpio_mode(13U, AVR_GPIO_OUTPUT);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `(void)avr_timer_ctc(AVR_TIMER0, 64U, 249U);` - Configures 0 in CTC mode with prescaler 64 and OCRnA TOP 249; event rate is 16 MHz / (64 x (249 + 1)) = 1000 Hz. The HAL disables this timer's interrupt sources and leaves flag polling to the caller.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `if ((TIFR0 & _BV(OCF0A)) != 0U)` - Contains Timer0 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `TIFR0 = _BV(OCF0A);` - Contains Timer0 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `++count;` - Advances the event/sample counter once for the condition just observed.
- `if (count == 500U)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `count = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `(void)avr_gpio_toggle(13U);` - Atomically toggles the selected pin output latch.

Algorithm: 16 MHz / 64 / (249+1) = 1000 compare matches per second. Polling can miss events if the main loop is blocked: a flag is not a counter.
Expected behaviour: L toggles every 500 compare matches, nominally 500 ms
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/gpio.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/gpio.h)
- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/gpio.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/gpio.c)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



### Timer1 one-second compare
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_gpio_mode(13U, AVR_GPIO_OUTPUT);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `(void)avr_timer_ctc(AVR_TIMER1, 256U, 62499U);` - Configures 1 in CTC mode with prescaler 256 and OCRnA TOP 62499; event rate is 16 MHz / (256 x (62499 + 1)) = 1 Hz. The HAL disables this timer's interrupt sources and leaves flag polling to the caller.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `if ((TIFR1 & _BV(OCF1A)) != 0U)` - Contains Timer1 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `TIFR1 = _BV(OCF1A);` - Contains Timer1 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `(void)avr_gpio_toggle(13U);` - Atomically toggles the selected pin output latch.

Algorithm: A 16-bit timer can count 62500 prescaled clocks. Timer0 cannot hold TOP=62499 because it is only eight bits.
Expected behaviour: L changes once per second: full LED cycle is two seconds
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/gpio.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/gpio.h)
- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/gpio.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/gpio.c)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



### Timer2 independent clock divider
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_gpio_mode(13U, AVR_GPIO_OUTPUT);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `(void)avr_timer_ctc(AVR_TIMER2, 128U, 124U);` - Configures 2 in CTC mode with prescaler 128 and OCRnA TOP 124; event rate is 16 MHz / (128 x (124 + 1)) = 1000 Hz. The HAL disables this timer's interrupt sources and leaves flag polling to the caller.
- `uint16_t count = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `if ((TIFR2 & _BV(OCF2A)) != 0U)` - Contains Timer2 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `TIFR2 = _BV(OCF2A);` - Contains Timer2 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `if (++count == 500U)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `count = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `(void)avr_gpio_toggle(13U);` - Atomically toggles the selected pin output latch.

Algorithm: Timer2 supports divide-by-128; Timer0 and Timer1 do not. At 16 MHz this yields 125 kHz and TOP=124 yields 1 kHz.
Expected behaviour: 500 compare events produce each LED transition
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/gpio.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/gpio.h)
- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/gpio.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/gpio.c)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



### PWM LED brightness
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_timer_pwm8(AVR_TIMER2, 64U, 0U, 0U);` - Selects 8-bit fast PWM, connects the chosen timer compare outputs, loads OCRnA/OCRnB and starts the prescaled clock. This takes ownership of that timer and its output pins.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `for (uint16_t duty = 1U; duty < 255U; ++duty)` - Repeats while `duty < 255U` is true; `uint16_t duty = 1U` sets the starting state and `++duty` advances it each pass.
- `OCR2A = (uint8_t)duty;` - Timer2 Compare A sets the match point or PWM duty value, depending on the selected mode.
- `_delay_ms(10);` - Waits for the stated interval; the foreground does no other work during this blocking delay.

Algorithm: Fast PWM frequency is 16 MHz / 64 / 256 = 976.5625 Hz. Average power changes with duty; perception is nonlinear. Endpoint OCR=0 has special pulse behavior.
Expected behaviour: External LED smoothly increases brightness then restarts
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



### SG90 servo pulse generation
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_timer1_pwm(8U, 40000U, 3000U);` - Configures Timer1 mode 14 with prescaler 8; ICR1 becomes 39999, giving a 20 ms frame, and OCR1A starts at 3000 ticks (1500 us high).
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `OCR1A = 2000U;` - Loads 2000 timer ticks for a 1 ms high pulse; Timer1 ticks at 0.5 us with the divide-by-8 prescaler.
- `_delay_ms(1000);` - Waits for the stated interval; the foreground does no other work during this blocking delay.
- `OCR1A = 3000U;` - Loads 3000 timer ticks for a 1.5 ms high pulse; this is the nominal servo centre position.
- `OCR1A = 4000U;` - Loads 4000 timer ticks for a 2 ms high pulse; the servo endpoint depends on the actual servo.

Algorithm: At divide-by-8 each tick is 0.5 us. 40000 ticks make a 20 ms frame. Servo angle and safe travel depend on the actual unit; do not force a mechanical stop.
Expected behaviour: Servo alternates approximately between 1, 1.5 and 2 ms pulse positions
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



### Timer1 hardware edge timestamp
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_uart_init(9600UL);` - Configures USART0 for asynchronous 8N1. It chooses a baud divisor and normal/double-speed mode, programs UBRR0/UCSR0A/UCSR0C, then enables receive and transmit in UCSR0B.
- `(void)avr_timer1_capture(8U, true);` - Starts Timer1 input capture at ICP1 (Uno D8), choosing the edge and prescaler. The application still owns any capture ISR and overflow accounting.
- `TIMSK1 = _BV(ICIE1);` - Controls Timer1 interrupt enables; only the selected event sources can request an interrupt.
- `sei();` - Enables global interrupts after the example has configured its sources and shared state.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `uint16_t copy = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `bool available = false;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `if (ready != 0U)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `copy = width;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `ready = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `available = true;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `if (available)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `(void)avr_uart_u16(copy);` - Formats an unsigned integer as decimal digits and sends them on USART0; it does not add a newline.
- `(void)avr_uart_put('\n', AVR_POLL_BUDGET);` - Waits for the USART transmit buffer within its poll budget, then writes the byte to UDR0.

Algorithm: ICR1 captures the counter at the hardware edge; interrupt latency does not shift that timestamp. One unsigned subtraction handles one wrap, not multiple wraps.
Expected behaviour: Prints high-pulse width in 0.5 us ticks
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`include/avr_hal/uart.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/uart.h)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)
- [`src/uart.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/uart.c)



## References

- [Timer HAL header](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h) and [implementation](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)
The current HAL headers and implementation are documented in the HAL Library README and API comments.
- [Timer0](timer0.md), [Timer1](timer1.md), [Timer2](timer2.md)
