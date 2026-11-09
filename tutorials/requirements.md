---
title: "Prerequisites And Equipment"
description: "Prepare the C knowledge, Uno hardware and software used throughout the course."
id: requirements
prerequisites: [welcome]
experiments: [01-blink-register]
validation: "Reviewed against Uno R3 documentation and the course Windows/PowerShell setup; electrical-safety advice checked against the linked device and board references"
---

# Prerequisites and Equipment

## What You Should Know

The lessons assume basic C: variables, integer expressions, `if` statements, loops, functions, arrays and header files. If you have written small programs but are unsure about pointers or bitwise operators, that is fine; the course introduces the parts needed for the labs.

You do not need to know electronics beforehand. Be ready to use a breadboard carefully, follow a wiring table and measure a voltage with a multimeter. Learn the difference between ground and a signal connection before adding external modules.

## Hardware for the First Labs

Start with:

- An Arduino Uno R3 with an ATmega328P and a known-good data USB cable.
- A computer with a current VS Code installation. The step-by-step setup and command examples in this course are written for Windows and PowerShell; other operating systems need equivalent compiler and terminal paths.
- A breadboard, jumper wires, a momentary push button and a few LEDs.
- Resistors such as 220–1,000 ohms for LEDs and 10 kilohms for simple pull-up or pull-down circuits.
- A digital multimeter for checking supply and signal voltages.

The onboard Uno LED is connected to D13, so it is useful for the first output test. External LEDs need a series resistor. The board's 5 V pin is a supply rail; it is not safe to assume every sensor module accepts 5 V. Check the module's data sheet or product documentation before connecting it.

An oscilloscope or logic analyser is helpful for timing and serial lessons, but is not required for the first builds. No external programmer is needed for normal bootloader uploads. ISP programming is a separate topic and should be attempted only with the correct target and configuration.

## Software and Accounts

The course uses MPLAB for VS Code with XC8 for bare-metal C builds. AVR GCC is also used for a second compiler check. Arduino IDE 2 and Arduino AVR Boards are used for selected comparison sketches. The [installation lesson](install-mplab.md) gives the Windows setup steps and a build checkpoint. The version table there records the authoring PC; it is an example configuration rather than a requirement to install those exact versions.

You can read Markdown lessons and build the examples without KiCad, Node.js or the authoring tools used to create the website. An Internet connection is useful for downloading software and opening the linked manufacturer documents. The reference PDFs in the local `Documents/` folder are optional study material and are not included in the public release by default.

## Before You Connect a Circuit

Check the Uno model and the component markings. With USB unplugged, compare the wiring against the table and schematic. Confirm the supply voltage and ground, then check the circuit again before applying power. Do not connect motors, relay coils or high-current loads directly to a GPIO pin; use a suitable driver circuit and supply.

## Readiness Check

What should you verify before connecting a sensor marked `VCC` to the Uno's 5 V rail?

**Answer:** Find the sensor or breakout's allowed supply range and the signal-voltage limits of its pins. `VCC` is only a label; it does not guarantee 5 V compatibility.

## Code Walkthrough


**Example code on GitHub:**
- [Register-level Blink (`01-blink-register`)](https://github.com/arnabdasbwn/atmega328p-labs/blob/main/examples/01-blink-register/main.c)

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


## References

- [Arduino Uno R3 documentation](https://docs.arduino.cc/hardware/uno-rev3/)
- [Arduino Uno R3 pinout](https://www.arnabkumardas.com/pinout/arduino-uno-r3/)
- [ATmega328P product page](https://www.microchip.com/en-us/product/ATMEGA328P)
