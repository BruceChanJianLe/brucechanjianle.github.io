---
title: Notes on Assembly Language
author: cjl
date: 2023-08-14 21:50:27 +0800
categories: [Programming]
tags: [programming]
---

> Just some notes on my assembly language learning journey.
> It could be a little messy, hopefully, it can be of help to future self and others.

Chips (CPUs) use instructions to manipulate binary data,
we call this `machine code`. It is often represented as hexadecimal.


Different manufacturers have their own set of instruction set.
The most popular set, is the Intel x86 chip instruction set.
There are also two commonly known instruction set, ARM chip, which is the most popular mobile phone chip; 
and Atmel chip, which appears in many small IoT type devices.
This learning journey will focus more on the Intel x86 chip.


CPU uses a set of registers, special areas of the chip, which are able to manipulate bits.
For example, instruction to add 28 to a register called ESP will be coded in machine language
as `83 C4 1C`. This is tricky for us - humans. Although many low level programs can write `machine code` directyly.
It is more common to use `mnemonics` to represent the various parts of instructions,
which is what is known as the `assembly language`.
The above `machine code` can be written in `assembly language` as `ADD ESP, 1C`.
- **Machine code**: `83 C4 1C`
- **Assembler**: `ADD ESP, 1C`

Therefore, a proper squence would be:
1. We code in assembly language, e.g. `ADD ESP, 1C`
1. A programme called Assembler, converts this mnemonic form back `machine code`, `83 C4 1C`
1. CPU executes it :D

[Fun fact](https://www.youtube.com/watch?v=fpnE6UAfbtU&ab_channel=CrashCourse): a `register` is a group of latches which holds a single number. The number of bits in a register is called its `width`.

Draw this diagram combine with the LK one (https://www.youtube.com/watch?v=Zr8WKIOIKsk&ab_channel=ALLABOUTELECTRONICS)
