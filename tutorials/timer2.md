---
title: "Timer/counter2 On The ATmega328P"
description: "Configure the eight-bit Timer2, calculate its distinct prescaler settings and learn the asynchronous clock precautions."
id: timer2
prerequisites: [clock, avr-architecture, interrupts, gpio-registers]
experiments: [68-timer2-register-ctc, 09-timer2-ctc, 11-pwm-led, 40-buzzer]
validation: "Synchronous prescalers, CTC setup and asynchronous update-busy flags reviewed against the ATmega328P documentation; register-level and HAL labs have GCC/XC8 builds; physical timing has not been measured."
---

# Timer/counter2 on the ATmega328P

## Learning Goals

Timer2 is an eight-bit timer with compare, PWM and optional asynchronous clock operation. This lesson shows how to calculate a synchronous CTC rate, use the Timer2-specific divider table and recognise the extra steps needed for an external watch crystal.

## The Timer2 Divider Table Is Different

In synchronous mode, Timer2 supports divisors of 1, 8, 32, 64, 128, 256 and 1024. Timer0 does not have the divide-by-32 or divide-by-128 selections. The clock-select bits (`CS22:0`) also use Timer2-specific encodings. Copying `CS` bits from Timer0 can therefore select the wrong clock or stop the timer.

In CTC mode 2, `OCR2A` is TOP. The compare period is:

```text
T_match = prescaler × (OCR2A + 1) / F_CPU
```

At 16 MHz, divide by 128 to get a 125 kHz timer clock, or 8 µs per tick. With `OCR2A = 124`, the timer counts 125 ticks per compare, so the event rate is `125,000 / 125 = 1,000 Hz`.

![How a timer compare match creates a periodic event](../assets/diagrams/timer-compare.svg)

## Register-Level Setup

The setup below selects CTC mode, clears the counter and pending flag, and then starts Timer2 at divide by 128. `OCIE2A` remains disabled; the foreground code polls `OCF2A`.

```c
TCCR2A = 0U;
TCCR2B = 0U;
TIMSK2 = 0U; /* This example polls; it owns Timer2's interrupt enables. */
TCNT2 = 0U;
OCR2A = 124U;
TIFR2 = _BV(OCF2A); /* W1C: write one to clear the pending compare flag. */
TCCR2A = _BV(WGM21); /* WGM22:0 = 0b010 selects CTC with OCR2A as TOP. */
TCCR2B = _BV(CS22) | _BV(CS20); /* CS22:0 = 0b101 selects divide by 128. */
```

Timer2 interrupt flags use write-one-to-clear behavior. Clear the flag with `TIFR2 = _BV(OCF2A);` rather than a read-modify-write such as `TIFR2 |= _BV(OCF2A);`.

## Asynchronous Timer2 and a Watch Crystal

Timer2 can run from a 32.768 kHz watch crystal on the TOSC1/TOSC2 pins while the CPU clock is stopped in Power-save sleep. This is useful for a low-power time base, but it is not enabled by the CTC setup above. Asynchronous operation uses `ASSR` and has separate register-update synchronization rules.

After enabling asynchronous operation, writes to `TCNT2`, `OCR2A`/`OCR2B` and `TCCR2A`/`TCCR2B` cross between clock domains. Check the corresponding update-busy flags in `ASSR` (`TCN2UB`, `OCR2AUB`/`OCR2BUB` and `TCR2AUB`/`TCR2BUB`) before assuming a write has reached the timer. Follow the datasheet's safe clock-switching sequence and disable Timer2 interrupts during the change. After power-up or a wake from Power-down/Standby, wait at least one second for the 32.768 kHz oscillator to stabilise before relying on the asynchronous timer. Do not use the synchronous prescaler calculation for a timer clocked from the watch crystal.

## Shared Timer Resources

Timer2 has two compare channels and output pins OC2A/D11 and OC2B/D3. PWM and tone generation also use its compare registers, mode bits and clock divider. If one feature configures Timer2 for CTC while another expects PWM, the later setup changes the first feature's behavior. Assign a single owner and document the selected mode and pins.

## Build and Observe the Lab

Lab 68 is the register-level version of the 1 kHz CTC exercise. Lab 09 uses the Timer HAL for the same event rate. Compare the Timer2 clock-select bits in both examples, then observe the D13 LED toggle after every 500 polled events, about every 500 ms. Lab 11 demonstrates PWM, and Lab 40 generates a tone; these are separate projects because they use Timer2 differently.

```powershell
python atmega328p-labs/tools/build.py 68-timer2-register-ctc --compiler gcc
python atmega328p-labs/tools/build.py 68-timer2-register-ctc --compiler xc8
python atmega328p-labs/tools/build.py 09-timer2-ctc --compiler gcc
python atmega328p-labs/tools/build.py 09-timer2-ctc --compiler xc8
```

- [Lab 68 register-level source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/68-timer2-register-ctc/main.c>)
- [Lab 09 HAL source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/09-timer2-ctc/main.c>)

These commands confirm the examples build with each compiler. They do not establish the LED timing or tone frequency on a physical Uno.

## Code Walkthrough


**Example code on GitHub:**
- [PWM LED brightness (`11-pwm-led`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/11-pwm-led/main.c)
- [Timer2 square-wave tone (`40-buzzer`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/40-buzzer/main.c)

### Timer2 Register-Level CTC Polling
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `uint16_t matches = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `PORTB &= (uint8_t)~_BV(PORTB5);` - Sets or clears the Port B output latch. When the pin is an output, this determines its output level; on an input, it controls the pull-up.
- `DDRB |= _BV(DDB5);` - Sets the Port B direction bits; a set bit selects output mode.
- `TCCR2A = 0U;` - Selects Timer2 waveform mode and compare-output behavior.
- `TCCR2B = 0U;` - Stops the timer clock while the remaining mode, counter and compare registers are prepared.
- `TIMSK2 = 0U;` - Controls Timer2 interrupt enables. A zero value leaves this example polling timer flags.
- `TCNT2 = 0U;` - Holds the live 8-bit Timer2 count; writing zero starts from a known count.
- `OCR2A = 124U;` - Timer2 Compare A sets the match point or PWM duty value, depending on the selected mode.
- `TIFR2 = _BV(OCF2A);` - Contains Timer2 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `TCCR2A = _BV(WGM21);` - Sets WGM21 in TCCR2A, selecting Timer2 CTC mode with OCR2A as TOP.
- `TCCR2B = _BV(CS22) | _BV(CS20);` - Sets Timer2 CS22:CS20 to 0b101, selecting its divide-by-128 prescaler.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `if ((TIFR2 & _BV(OCF2A)) != 0U)` - Contains Timer2 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `++matches;` - Advances the event/sample counter once for the condition just observed.
- `if (matches == 500U)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `matches = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `PINB = _BV(PINB5);` - Writing a one to a PINB bit toggles that pin's PORTB output latch on the ATmega328P.

Algorithm: Directly configures synchronous Timer2 CTC mode at 16 MHz / 128 / (124 + 1) = 1 kHz. Timer2 uses a distinct prescaler encoding from Timer0.
Expected behaviour: The onboard LED toggles every 500 compare events, nominally every 500 ms
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)


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



### Timer2 square-wave tone
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_timer_ctc(AVR_TIMER2, 64U, 124U);` - Configures 2 in CTC mode with prescaler 64 and OCRnA TOP 124; event rate is 16 MHz / (64 x (124 + 1)) = 2000 Hz. The HAL disables this timer's interrupt sources and leaves flag polling to the caller.
- `DDRB |= _BV(DDB3);` - Sets the Port B direction bits; a set bit selects output mode.
- `TCCR2A |= _BV(COM2A0);` - Selects Timer2 waveform mode and compare-output behavior.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.

Algorithm: A compare event toggles OC2A, so two compare periods make one waveform period: 16 MHz/(2*64*125)=1 kHz.
Expected behaviour: Produces nominally 1 kHz tone
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/timer.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/timer.h)
- [`src/timer.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/timer.c)



## Check Your Understanding

At 16 MHz with Timer2 divided by 32 and `OCR2A = 249`, what compare rate should you expect?

**Answer:** `16,000,000 / (32 × 250) = 2,000 Hz`.

## References

- Microchip, [ATmega328P documentation](https://www.microchip.com/en-us/product/ATMEGA328P), Timer/Counter2 chapter, including asynchronous operation and `ASSR` update-busy flags.
- [Timer2 CTC source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/09-timer2-ctc/main.c>), [PWM LED](pwm-led.md) and [tone generation](buzzer.md).
