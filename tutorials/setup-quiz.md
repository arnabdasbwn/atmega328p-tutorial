---
title: "Setup Checkpoint"
description: "Review the compiler, board, COM port and safe upload steps before beginning the peripheral labs."
id: setup-quiz
prerequisites: [install-mplab, arduino-drivers, upload-usb, upload-isp]
experiments: [01-blink-register]
validation: "Answers cross-checked against the setup, upload and ATmega328P lessons; each answer includes its reasoning"
---

# Setup Checkpoint

## Questions

Try these without looking at the answers first.

1. What does the `F_CPU` definition do, and does it change the Uno's physical clock?
2. Your build succeeds, but no COM port appears in Windows. Which problem should you investigate first?
3. Why does an ordinary application upload use the Uno bootloader instead of **Burn Bootloader**?
4. Which target signature should a correctly connected ATmega328P report?
5. What is the difference between a compile result, an upload result and a physical observation?

## Answers

1. `F_CPU` tells code and libraries the frequency to assume for timing calculations. It does not change the oscillator or program the clock fuses.
2. Check the data cable, USB interface and matching device driver in Device Manager. Compilation does not require a board or serial port.
3. The bootloader accepts application firmware over serial. **Burn Bootloader** is a separate operation that can erase Flash and configure fuses before writing bootloader code.
4. `1E 95 0F`. If the result is different or unstable, stop and verify the device and wiring before writing anything.
5. A compile result shows that the selected toolchain built an image. An upload result shows that the selected programming path reported acceptance. A physical observation shows what the connected hardware actually did; it should include the method and conditions.

## Continue Learning

Build the [first register-level Blink](first-blink.md), review the [USB upload procedure](upload-usb.md), and keep the build, programming and hardware evidence separate in your notes.

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
