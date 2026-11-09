---
title: "Course Introduction And Learning Path"
description: "Learn how C instructions control real hardware on an Arduino Uno R3."
id: welcome
prerequisites: []
experiments: []
validation: "Reviewed learning path, prerequisites, claims and linked destinations; orientation lesson, no physical test required"
---

# Course Introduction and Learning Path

An Arduino Uno R3 is both a convenient development board and a small computer you can understand in detail. Its ATmega328P executes instructions, stores variables in SRAM and exposes peripheral registers. This course connects those ideas: you will calculate a timer period, write the corresponding register values, and explain the signal you expect to measure.

## What You Need to Know

Basic C functions, loops, arrays and conditionals are assumed. You do not need previous register-level experience. The embedded C lessons explain the differences that matter on an 8-bit target: integer widths, hardware registers, interrupts and separate program/data memories.

Use an **Uno R3 with ATmega328P**. Uno R4 and boards based on ATmega328PB are different targets, even if their connectors look familiar. The reference board clock is 16 MHz and its GPIO logic is nominally 5 V. Always check the voltage requirements of an attached module.

## How to Study

1. Complete setup and build the [first Blink example](first-blink.md).
2. Study the CPU, memory and embedded C foundations alongside small programs.
3. For each peripheral, follow concept, registers, direct-register example and HAL example in that order.
4. Predict an observation before running a lab. Record the measured result and explain differences.
5. Complete the exercises before reading their answers.

Register-level code makes the hardware decisions visible. The HAL packages those decisions into reusable functions. Selected Arduino sketches show another way to express the same behavior. Neither approach guarantees good engineering by itself: understanding ownership, timing and failure behavior matters in both.

## A Concrete Example

A 16 MHz CPU clock has a 62.5 ns period. That does **not** mean every C statement takes 62.5 ns: a statement may compile into several instructions, some instructions take multiple cycles, and interrupts can add latency. We will inspect compiler output and distinguish calculated timing from measured timing.

## Reading Validation Labels

“Build passed” means the compiler and linker accepted a program. “Simulation passed” means a named model satisfied specified checks. “Hardware unverified” means no physical test has been recorded. A sensor example may compile perfectly while the wrong breakout wiring prevents it from working; these labels deliberately describe different evidence.

## Downloads and Source

The local course checkout contains the atmega328p-hal and atmega328p-labs repositories as pinned submodules. Once their public remote URLs are configured, students will be able to use `git clone --recurse-submodules` or download the coordinated release ZIP. The repositories are not published yet. You only need website authoring tools if you intend to rebuild the website; firmware builds use the tools explained in setup.

## Exercise

Why is a successful compile insufficient evidence that an LED will blink?

**Answer:** the target board, clock, pin mapping, wiring and flashed image can be wrong independently of C syntax. A working compiler checks only part of that chain. The first lab therefore specifies a board, a pin, a clock and an expected one-second waveform.

## References

For current device details, use the [Microchip ATmega328P documentation](https://www.microchip.com/en-us/product/ATMEGA328P). Check the document listed for your exact device and package; board-level pin labels and circuitry are covered by the [Uno R3 pinout](https://www.arnabkumardas.com/pinout/arduino-uno-r3/).
