---
title: "Use A Timer For System Delays"
description: "Replace a long blocking delay with elapsed-time checks using a periodic timer tick."
id: timer-delay
prerequisites: [timer0, interrupts, volatile-atomic, state-machines]
experiments: [10-scheduler]
validation: "elapsed-time wrap guidance and Timer0 ownership reviewed against the scheduler and tick HAL; GCC/XC8 builds pass; hardware timing is not measured"
---

# Use a Timer for System Delays

![Timer clock, prescaler, counter and compare event with a pending flag for polling or an interrupt](../assets/diagrams/timer-compare.svg)

## A Delay Stops Foreground Work

A busy delay keeps the CPU in a wait loop until time has passed. That is simple for a tiny demonstration, but the foreground cannot scan buttons, update a display or process a serial command during that wait. A timer tick lets the application check elapsed time and continue doing other bounded work between checks.

The course tick driver uses Timer0 compare A. At 16 MHz with a divide-by-64 prescaler, `OCR0A=249` produces a nominal 1 ms interrupt. The ISR increments a shared millisecond counter; foreground code reads an atomic snapshot using `avr_tick_ms()`.

## Check Elapsed Time Without Waiting

```c
uint32_t now = avr_tick_ms();
if ((uint32_t)(now - previous_ms) >= interval_ms)
{
    previous_ms = now;
    update_output();
}
```

Unsigned subtraction continues to work across counter wrap for intervals below half the counter range, provided the foreground checks often enough. `previous_ms = now` schedules from the time the event was noticed; adding the interval to the previous deadline instead preserves phase but needs a policy for missed deadlines.

## Timer Ownership and Accuracy

The `avr_tick_init()` driver owns Timer0 and `TIMER0_COMPA_vect`; it cannot coexist with the Arduino core timebase or another Timer0 owner. Interrupts must be enabled by the application after setup. The tick is a software timebase, not a calibrated real-time clock, and long periods with interrupts disabled can lose compare events.

Build the [cooperative scheduler lab](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/10-scheduler/main.c>) with XC8 or AVR GCC. Its LED and UART examples illustrate independent periodic work without a delay loop.

## Exercise with Answer

Why does the tick ISR only update a counter instead of printing the elapsed time?

**Answer:** Printing can wait for UART hardware and take much longer than the tick interval. Keeping the ISR short limits interrupt latency; foreground code can format and send the message later.

## Code Walkthrough

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



## References

- [Timer0 and timer ownership](timer0.md)
- [Volatile and atomic access](volatile-atomic.md)
- [State machines and cooperative scheduling](state-machines.md)
- [Scheduler source](<https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/10-scheduler/main.c>)
