---
title: "Why Learn AVR And Embedded C?"
description: "See what register-level firmware teaches and how the Uno makes small computer systems visible."
id: motivation
prerequisites: [welcome]
experiments: [01-blink-register, 03-button, 14-uart-echo]
validation: "Reviewed against the Uno R3 and Microchip ATmega328P resources; links to maintained register-level, button and UART labs checked; no hardware behaviour is claimed here"
---

# Why Learn AVR and Embedded C?

## Objectives and Prerequisites

This lesson explains what you can learn from a small 8-bit microcontroller and how the course builds those skills. You should be comfortable with basic C statements, functions, loops and variables. The first lessons introduce the hardware ideas as they are needed.

## Learn by Connecting Code to a Pin

On a computer, a statement such as `count += 1;` changes a value in memory. On an ATmega328P, a C program can also change a register that controls an Uno pin. The LED lab gives a visible result; a button lab adds an input; the UART lab sends bytes to a computer. Each exercise connects a C operation to a measurable effect.

You will practise reading a datasheet, translating a board pin name to a port bit, calculating a timer setting, building the firmware and checking the result. These are useful habits for many microcontrollers. The register names and peripheral details are specific to the ATmega328P, so do not copy them blindly to a different chip.

## A Small Target Makes Trade-Offs Clear

The Uno R3 has 32 KB of program Flash, 2 KB of SRAM and a 16 MHz ATmega328P. That is enough for useful sensor and control exercises, but small enough that memory use, integer widths and timing decisions are easy to observe. For example, a 100-element array of 16-bit integers uses 200 bytes of SRAM before accounting for stack, other variables or buffers:

`100 elements × 2 bytes per element = 200 bytes`

That calculation helps explain why embedded programs need a memory budget. It does not mean every project needs manual optimisation; it means you should measure the resources your design uses.

## What This Course Does and Does Not Claim

Arduino is a convenient board ecosystem and Arduino sketches are useful for quick experiments. Register-level C exposes more of the device, while the course HAL shows how to wrap repeated operations behind a documented interface. Each method has trade-offs. A framework saves time; direct register work helps you understand ownership and timing.

The Uno is a teaching target, not a model for every current product. Newer devices may have more memory, different peripherals and different toolchains. The transferable skill is learning to identify the exact part and verify its behaviour from primary documentation.

## Code Walkthrough


**Example code on GitHub:**
- [Register-level Blink (`01-blink-register`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/01-blink-register/main.c)
- [Debounced push button (`03-button`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/03-button/main.c)
- [Bounded UART echo (`14-uart-echo`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/14-uart-echo/main.c)

### Register-level Blink
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `PORTB &= (uint8_t)~_BV(PORTB5);` - Sets or clears the Port B output latch. When the pin is an output, this determines its output level; on an input, it controls the pull-up.
- `DDRB |= _BV(DDB5);` - Sets the Port B direction bits; a set bit selects output mode.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `PINB = _BV(PINB5);` - Writing a one to a PINB bit toggles that pin's PORTB output latch on the ATmega328P.
- `_delay_ms(500);` - Waits for the stated interval; the foreground does no other work during this blocking delay.

Algorithm: Pin direction and output latch are separate hardware state.
Expected behaviour: L toggles every 500 ms
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)


### Debounced push button
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `avr_debounce_t button;` - Use the linked public header for this function's parameters, register ownership, blocking behavior and status values.
- `bool level = true;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `bool led = false;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `(void)avr_gpio_mode(2U, AVR_GPIO_INPUT_PULLUP);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `(void)avr_gpio_mode(13U, AVR_GPIO_OUTPUT);` - Maps the Uno header pin to its AVR port, sets its DDR direction bit and prepares the PORT latch for the requested initial level or pull-up.
- `avr_tick_init();` - Configures the course millisecond tick interrupt source; it does not enable global interrupts itself.
- `avr_debounce_init(&button, true, 0UL);` - Seeds the stable/candidate input state and timestamp so the next transition is timed consistently.
- `sei();` - Enables global interrupts after the example has configured its sources and shared state.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `(void)avr_gpio_read(2U, &level);` - Reads the electrical level through PINx without changing direction or the output latch.
- `if (avr_debounce_update(&button, level, avr_tick_ms(), 20U) && !button.stable)` - This condition chooses the next path based on the API result. Accepts a changed input only after it remains stable for the requested interval, then reports that accepted change once.
- `led = !led;` - Stores this intermediate or output value for the next algorithm step. Check the units and range represented by each operand.
- `(void)avr_gpio_write(13U, led);` - Changes the pin's PORT output latch; for an input pin this selects its pull-up state.

Algorithm: A candidate input must remain unchanged for 20 ms before becoming stable. No delay occurs inside an ISR.
Expected behaviour: Each press toggles L once; holding the button does not repeat
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/gpio.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/gpio.h)
- [`include/avr_hal/tick.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/tick.h)
- [`lib/drivers/debounce.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/lib/drivers/debounce.c)
- [`lib/drivers/debounce.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/lib/drivers/debounce.h)
- [`lib/interrupts/tick.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/lib/interrupts/tick.c)
- [`src/gpio.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/gpio.c)



### Bounded UART echo
The example's `main()` configures its peripherals, runs the algorithm and reports or displays the result. These source operations are listed in execution order:

- `(void)avr_uart_init(9600UL);` - Configures USART0 for asynchronous 8N1. It chooses a baud divisor and normal/double-speed mode, programs UBRR0/UCSR0A/UCSR0C, then enables receive and transmit in UCSR0B.
- `for (;;)` - Repeats the foreground task indefinitely; each pass checks inputs, advances the calculation or services the peripheral shown below.
- `avr_status_t status = avr_uart_get(&byte, AVR_POLL_BUDGET);` - Use the linked public header for this function's parameters, register ownership, blocking behavior and status values.
- `if (status == AVR_OK)` - Only enters the following block when the operation reports success.
- `(void)avr_uart_put(byte, AVR_POLL_BUDGET);` - Waits for the USART transmit buffer within its poll budget, then writes the byte to UDR0.

Algorithm: One 8N1 frame contains ten bits. At 9600 baud, a frame takes approximately 1.042 ms; the application must service reception before hardware buffering overflows.
Expected behaviour: Each received byte is echoed once
**HAL source files in this repository:** [ATmega328P HAL on GitHub](https://github.com/arnabdasbwn/atmega328p-hal)

- [`include/avr_hal/uart.h`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/include/avr_hal/uart.h)
- [`src/uart.c`](https://github.com/arnabdasbwn/atmega328p-hal/blob/main/src/uart.c)



## Check Your Understanding

Why does learning to set a GPIO register help with later peripheral lessons?

**Answer:** It shows how firmware reaches hardware through a defined interface. Timers, ADC, serial communication and other peripherals also use registers, but each has its own configuration and status rules. The common habit is to read the device documentation, set up the peripheral deliberately and verify the resulting signal or data.

## References

- [ATmega328P product page and device documentation](https://www.microchip.com/en-us/product/ATMEGA328P)
- [Course introduction and learning path](welcome.md)
- [ATmega328P specifications and practical limits](atmega328p-specs.md)
