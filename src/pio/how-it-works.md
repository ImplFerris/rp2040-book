# How PIO works in Raspberry Pi Pico?

> **TL;DR**
>
> In PIO, you write a small program that describes exactly how GPIO pins should change over time. This program does not run on the main CPU. Instead, it runs on a tiny hardware engine inside the RP2040, called a state machine. Once started, the program executes repeatedly with precise timing and directly controls the pins on its own. The main CPU only sets things up and exchanges data when needed.

The RP2040 has two PIO blocks, and they are identical.  Each one has four state machines and its own memory where the program is stored.

<div class="image-with-caption" style="text-align:center;">
    <img src="./images/PIO blocklevel diagram.png" alt="PIO blocklevel diagram" style="width:70%;height:auto; display:block; margin:auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Single PIO block (from Chapter 3.1 of the RP2040 datasheet)</div>
</div>

## State Machine

A state machine is a small processor that runs a simple program in a loop. It is not as powerful as the main CPU. But it can do one thing well: run a short set of steps, over and over, with exact timing. 

> [!Tip]
> If you have never heard the phrase "state machine" before, it basically describes a system that operates by moving between a fixed set of states based on defined conditions. For example, an LED controller might switch between ON and OFF states based on a timer.

So across the whole chip you have eight state machines in total. PIO block 0 has state machines 0 through 3, and PIO block 1 also has its own state machines 0 through 3. Each one runs its own program. They all run at the same time. They do not wait for each other unless you tell them to.

Each PIO state machine can read from and write to the GPIO pins.

We write programs for a state machine using a small assembly language designed specifically for the RP2040 PIO. This instruction set is intentionally minimal and contains only 9 instructions. You can refer to the datasheet for full details, but we will take a quick overview of them next.

## Instruction memory

Each PIO block has a small instruction memory that can hold up to 32 instructions.

All four state machines in one PIO block share this same memory. If you are running two programs in the same block, they both have to fit inside those 32 instructions together. In practice this is not usually a problem. PIO programs are short. But it is something to keep in mind.

<div class="image-with-caption" style="text-align:center;">
    <img src="./images/PIO Instruction Memory shared with State Machine.svg" alt="PIO Instruction Memory shared with State Machine" style="width:70%;height:auto; display:block; margin:auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">PIO Instruction Memory shared with State Machine</div>
</div>

Each state machine keeps track of where it is in the memory on its own. So state machine 0 might be running instructions 0 through 5, and state machine 1 might be running instructions 6 through 12. They are reading from the same memory, but they are at different places in it.
