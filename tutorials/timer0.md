---
title: "Timer/counter0 On The ATmega328P"
description: "Configure the eight-bit Timer0 in CTC mode, calculate its compare rate and understand Arduino timer ownership."
id: timer0
prerequisites: [clock, avr-architecture, interrupts, gpio-registers]
experiments: [66-timer0-register-ctc, 07-timer0-ctc, 10-scheduler]
validation: "Worked compare calculations and W1C flag access reviewed against the ATmega328P documentation; register-level and HAL labs have GCC/XC8 builds; physical timing has not been measured."
---

# Timer/counter0 on the ATmega328P

## Learning Goals

After this lesson, you should be able to configure Timer0 in Clear Timer on Compare Match (CTC) mode, calculate a compare period from the clock and prescaler, and explain why a polled flag can lose events.

## Timer0 in Ctc Mode

Timer/Counter0 is an eight-bit counter. Its count can range from 0 to 255. In CTC mode, the counter counts upward from BOTTOM (zero) until it matches `OCR0A`; the match sets `OCF0A` and clears the counter on the next timer clock. The compare flag remains set until software clears it or the corresponding interrupt vector runs.

![How a timer compare match creates a periodic event](../assets/diagrams/timer-compare.svg)

For the normal synchronous clock source, Timer0 supports prescalers of 1, 8, 64, 256 and 1024. Its maximum CTC count is 256 timer clocks because `OCR0A` can be 255 and counting includes zero. The match rate is:

```text
f_match = F_CPU / (prescaler × (OCR0A + 1))
```

At 16 MHz with a divide-by-64 prescaler and `OCR0A = 249`, one timer tick is `64 / 16,000,000 = 4 µs`. A compare takes 250 ticks, or 1 ms, so the compare rate is 1 kHz. The `+1` matters because the timer visits zero as well as the TOP value.

## Register-Level Setup

The following setup selects CTC mode, loads TOP and starts the timer. It leaves compare interrupts disabled, so the foreground code must poll `OCF0A`.

```c
TCCR0A = 0U;
TCCR0B = 0U;                 /* Stop the timer while changing its setup. */
TIMSK0 = 0U;                 /* This example polls; it owns Timer0's interrupt enables. */
TCNT0 = 0U;
OCR0A = 249U;
TIFR0 = _BV(OCF0A);          /* Clear a pending flag by writing one. */
TCCR0A = _BV(WGM01);         /* WGM02:0 = 0b010 selects CTC with OCR0A as TOP. */
TCCR0B = _BV(CS01) | _BV(CS00); /* CS02:0 = 0b011 selects divide by 64. */
```

When `OCF0A` is set, clear only that flag with a direct write:

```c
if ((TIFR0 & _BV(OCF0A)) != 0U)
{
    TIFR0 = _BV(OCF0A);
    /* Record one observed compare event here. */
}
```

Timer interrupt flags use write-one-to-clear (W1C) behavior. Avoid `TIFR0 |= _BV(OCF0A)`: the read-modify-write can write ones back to other flags that were already set and clear them too. The [GPIO register lesson](gpio-registers.md) discusses why peripheral registers need the right access pattern.

## Polling, Interrupts and Missed Events

A hardware flag is one bit, not a queue. If the program takes longer than one compare period before checking `OCF0A`, several matches may occur while the flag is already one. The program then sees one pending event and cannot tell how many matches passed. Polling is useful for a small demonstration, but it is not a reliable way to count events when other work can block the loop.

An interrupt can let the timer notify the CPU, but the interrupt service routine should do only bounded work, such as incrementing a counter. It should not print to UART, wait for a device, or perform a long calculation. See [Interrupts](interrupts.md) and [Timer Interrupts](timer-interrupt.md) before enabling `OCIE0A` and defining `TIMER0_COMPA_vect`.

## Timer Ownership on the Uno

The Arduino AVR core uses Timer0 for timekeeping functions such as `millis()` and `delay()`. This bare-metal configuration changes Timer0's mode, divider and compare registers, so it cannot safely share a program with those core services. The course HAL labs are separate bare-metal projects. Do not copy their timer setup into an Arduino sketch and expect Arduino time functions to remain correct.

## Build and Observe the Lab

Lab 66 is the register-level version of this exercise. Lab 07 uses the Timer HAL to configure the same 1 kHz CTC event. Compare the two `main.c` files: the timer registers are configured directly in Lab 66, while Lab 07 calls the shared `avr_timer_ctc()` API. Both toggle D13 after 500 observed matches, giving a nominal 500 ms between LED transitions. The LED's full on/off cycle is about one second.

```powershell
python atmega328p-labs/tools/build.py 66-timer0-register-ctc --compiler gcc
python atmega328p-labs/tools/build.py 66-timer0-register-ctc --compiler xc8
python atmega328p-labs/tools/build.py 07-timer0-ctc --compiler gcc
python atmega328p-labs/tools/build.py 07-timer0-ctc --compiler xc8
```

Lab 10 demonstrates a scheduler tick. These builds validate compilation; no external timing measurement is claimed. On hardware, a slow LED change confirms gross operation only. A scope or logic analyser is needed to measure the compare-derived timing accurately.

- [Lab 66 register-level source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/66-timer0-register-ctc/main.c>)
- [Lab 07 HAL source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/07-timer0-ctc/main.c>)

## Code Walkthrough


**Example code on GitHub:**
- [Interrupt tick and cooperative tasks (`10-scheduler`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/10-scheduler/main.c)

### Timer0 Register-Level CTC Polling
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `uint16_t matches = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `PORTB &= (uint8_t)~_BV(PORTB5);` - Sets or clears the Port B output latch. When the pin is an output, this determines its output level; on an input, it controls the pull-up.
- `DDRB |= _BV(DDB5);` - Sets the Port B direction bits; a set bit selects output mode.
- `TCCR0A = 0U;` - Clears Timer0 mode/output bits before setting the intended mode.
- `TCCR0B = 0U;` - Stops the timer clock while the remaining mode, counter and compare registers are prepared.
- `TIMSK0 = 0U;` - Controls Timer0 interrupt enables. A zero value leaves this example polling timer flags.
- `TCNT0 = 0U;` - Holds the live 8-bit Timer0 count; writing zero starts from a known count.
- `OCR0A = 249U;` - Loads TOP=249. With a 16 MHz clock divided by 64, the compare rate is 16 MHz / 64 / (249 + 1) = 1 kHz.
- `TIFR0 = _BV(OCF0A);` - Contains Timer0 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `TCCR0A = _BV(WGM01);` - Sets WGM01 in TCCR0A, selecting Timer0 CTC mode with OCR0A as TOP.
- `TCCR0B = _BV(CS01) | _BV(CS00);` - Sets CS01:CS00 to 0b011, starting Timer0 with a divide-by-64 prescaler.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `if ((TIFR0 & _BV(OCF0A)) != 0U)` - Contains Timer0 event flags. These flags are write-one-to-clear: assign the mask directly and do not use |=.
- `++matches;` - Advances the event/sample counter once for the condition just observed.
- `if (matches == 500U)` - Tests the stated condition; the following block handles the matching case and the other path is skipped.
- `matches = 0U;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `PINB = _BV(PINB5);` - Writing a one to a PINB bit toggles that pin's PORTB output latch on the ATmega328P.

Algorithm: Directly configures Timer0 in CTC mode at 16 MHz / 64 / (249 + 1) = 1 kHz. A pending compare flag is one bit and can merge events if the foreground is delayed.
Expected behaviour: The onboard LED toggles every 500 compare events, nominally every 500 ms
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)


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



### Interrupt tick and cooperative tasks
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `uint32_t led_start = 0UL, report_start = 0UL;` - Clears the state/counter after its threshold is reached so the next interval starts from zero.
- `(void)avr_gpio_mode(13U, AVR_GPIO_OUTPUT);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `(void)avr_uart_init(9600UL);` - Configures USART0 for asynchronous 8N1. It chooses a baud divisor and normal/double-speed mode, programs UBRR0/UCSR0A/UCSR0C, then enables receive and transmit in UCSR0B.
- `avr_tick_init();` - Configures the course millisecond tick interrupt source; it does not enable global interrupts itself.
- `sei();` - Enables global interrupts after the example has configured its sources and shared state.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `uint32_t now = avr_tick_ms();` - Use the linked public header for this function's parameters, register ownership, blocking behavior and status values.
- `if (avr_elapsed(now, led_start, 500UL))` - This branch runs only after the requested interval has elapsed. Uses unsigned subtraction to test elapsed time safely across a 32-bit tick wraparound.
- `led_start = now;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `(void)avr_gpio_toggle(13U);` - Atomically toggles the selected pin output latch.
- `if (avr_elapsed(now, report_start, 1000UL))` - This branch runs only after the requested interval has elapsed. Uses unsigned subtraction to test elapsed time safely across a 32-bit tick wraparound.
- `report_start = now;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `(void)avr_uart_u16((uint16_t)(now / 1000UL));` - Formats an unsigned integer as decimal digits and sends them on USART0; it does not add a newline.
- `(void)avr_uart_put('\n', AVR_POLL_BUDGET);` - Waits for the USART transmit buffer within its poll budget, then writes the byte to UDR0.

Algorithm: The ISR increments a tick; foreground code snapshots it atomically. Unsigned subtraction remains valid across wraparound for the documented interval range.
Expected behaviour: LED toggles twice per second while elapsed seconds print independently
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/gpio.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/gpio.h)
- [`include/avr_hal/tick.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/tick.h)
- [`include/avr_hal/uart.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/uart.h)
- [`lib/interrupts/tick.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/lib/interrupts/tick.c)
- [`src/gpio.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/gpio.c)
- [`src/uart.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/uart.c)



## Check Your Understanding

With `F_CPU = 16 MHz`, a divide-by-64 prescaler and `OCR0A = 199`, what compare rate do you expect?

**Answer:** `16,000,000 / (64 × 200) = 1,250 Hz`. The period is 800 µs.

## References

- Microchip, [ATmega328P documentation](https://www.microchip.com/en-us/product/ATMEGA328P), Timer/Counter0 chapter. The lesson uses the synchronous prescaler and CTC tables for the ATmega328P, not another AVR variant.
- [Timer0 CTC source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/07-timer0-ctc/main.c>) and [Timer HAL](timer-hal.md).
- [Arduino `millis()` reference](https://docs.arduino.cc/language-reference/en/functions/time/millis/) and [`delay()` reference](https://docs.arduino.cc/language-reference/en/functions/time/delay/), which depend on the Arduino core timebase.
