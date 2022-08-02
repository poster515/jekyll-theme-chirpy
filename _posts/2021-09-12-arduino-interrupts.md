---
title: Arduino Timer Interrupts
author: Joe Post
date: 2021-09-12 14:10:00 +0800
categories: [Blogging, Tutorial]
tags: [c++, programming, arduino, interrupts]
---

## Foreword

In this blog, we're going to explore a fundamental aspect of the Arduino architecture: the timer interrupt. Interrupts are an entire topic in and of themselves, but today we're focusing specifically on timer interrupts. This is for several reasons:

1. Timer interrupts are a subset of interrupts and should be logically extendable to other interrupts,
2. Timer interrupts can provide a very useful, precision time measurement utility for a variety of applications, and
3. I happen to find timer interrupts particularly interesting since they involve fundamental portions of the Atmel architecture.

This blog assumes you have an Arduino board (I used an Arduino Due), and the Arduino software installed on your computer. I will not be covering the exact steps to compile and upload the code to your board; it is assumed that you have done this before. 


## Timer Architecture
Before we dive into the code, we need to understand what _exactly_ what the timer infrastructure is within the Atmel chipset of interest. Again, I'm using an Arduino Due, but all you need to do is consult your specific chiset's user guide to find out the equivalent registers and code. 

The Arduino Due uses an Atmel SAM3X8E processor, which is much more capable than the Atmega-based line of other Arduino boards. The processor uses an 84 MHz clock, uses a Thumb-2 instruction set architecture (ISA) subset consisting of all base Thumb-2 instructions, 16-bit and 32-bit, a Harvard processor architecture enabling simultaneous instruction fetch with data load/store, three-stage pipeline, single cycle 32-bit multiply, hardware divide, and several other features.


The SAM3X83 has several internal timer/counter (TC) modules capable of generating interrupts. Page 38 of the [datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11057-32-bit-Cortex-M3-Microcontroller-SAM3X-SAM3A_Datasheet.pdf) documents the following peripheral identifiers related to the timer module interrupts:

| *Instance ID* | *Instance Name* | *NVIC Interrupt* | *PMC Clock Control* | *Description* |
| ------------- | ------------- | ------------- | ------------- | ------------- | 
| 27 | TC0 | X | X | Timer Counter Channel 0 | 
| 28 | TC1 | X | X | Timer Counter Channel 1 | 
| 29 | TC2 | X | X | Timer Counter Channel 2 | 
| ... | ... | ... | ... | ... | 
| 35 | TC8 | X | X | Timer Counter Channel 8 | 

Ok there are quite a few new acronyms above. Let's break these down. The TC modules each manage some sort of timing functionality - they can count up, down, issue interrupts, toggle general purpose input/outputs (GPIOs). These interrupts and associated peripheral identifiers are used by the Nested Vector Interrupt Controller (NVIC) to manage interrupts. The NVIC provides up to 16 interrupt priority levels. 

As far as system-level views of interrupts are concerned, there are some core registers that are associated with them.

## Use Cases

To be continued!

## Conclusion
We covered quite a few topics in this intro to C++ and general workflow. There are so, so many more interesting concepts to cover such as polymorphism, templates, threads, and many more. I may make more tutorials on these topics in the future. As usual, thanks for reading!

## Learn More

For more knowledge about Jekyll posts, visit the [Jekyll Docs: Posts](https://jekyllrb.com/docs/posts/).

