# Scratch Registers

So far, all of our delays were created using `nop`. That works, but it wastes a lot of instructions. This is where scratch registers become useful.

A scratch register is a small temporary register used to hold intermediate values while a program runs. In PIO, there are two such registers named X and Y. They hold temporary values that the program can modify as it runs.

You can load a value into X or Y, decrement it, and jump based on conditions. This makes them useful for building loops.

## Using X as a delay counter

A common pattern is to use the X (or Y) register as a counter.

We first load a value into X, then decrement it in a loop until it reaches zero.

```rust
set x, 31           ; 1 cycle
delay:
    nop [5]         ; 6 cycles per iteration  
    jmp x--, delay ; decrement X and jump (1 cycle)
```

> [!Note]
> When using the `pio_asm!` macro, trying to load a value larger than 31 (for example `set x, 32`) will result in a compile-time error such as:
> `SET argument out of range`.

Here, X starts at 31 and counts down to zero. Each time the `jmp x--, delay` instruction executes, the jump condition is evaluated first. If X is nonzero, execution jumps back to the label, with X being decremented as part of the same instruction. When X is zero at the test, the jump is not taken and the loop exits.

Although the X and Y registers are 32-bit wide, the `set` instruction can only load immediate values from 0 to 31. With `jmp x--`, the loop executes exactly the number of times loaded into X. With X set to 31, it results in 31 loop iterations.

The nop [5] consumes 6 cycles per iteration, and `jmp x--` consumes 1 cycle, for a total of 7 cycles per iteration. The loop therefore introduces a delay of 218 cycles, including the preceding `set x, 31` instruction.

## Using scratch registers in the blinky program

Now let us apply this idea to our LED blink example.

Earlier, we created delays by writing many `nop` instructions back to back. That approach works, but it wastes instruction memory. We can replace those repeated `nop` instructions with a small loop that uses a scratch register.

Below is the same blinky program rewritten to use the X register as a delay counter.

```rust
set pindirs, 1          ; Set pin direction to output (runs once)

.wrap_target
    set pins, 1 [31]    ; Set pin HIGH (32 cycles)

    set x, 31           ; Load delay counter (1 cycle)
delay_high:
    nop [5]             ; 6 cycles
    jmp x--, delay_high ; 1 cycle
    
    set pins, 0 [31]    ; Set pin LOW (32 cycles)

    set x, 31           ; Load delay counter (1 cycle)
delay_low:
    nop [5]             ; 6 cycles
    jmp x--, delay_low  ; 1 cycle

.wrap
```

> [!Tip]
> Try changing the value loaded into X or the delay modifier on the `nop` instruction.
> Observe how these changes affect the output timing.

Each loop iteration executes a `nop` instruction to consume cycles, then decrements X and repeats until the counter reaches zero. This creates a longer delay while using only a few instruction slots.

After the delay, the pin is set LOW and the same process is repeated for the LOW state.

Compared to the earlier version with many `nop` instructions, this version uses fewer instruction slots and is easier to read and maintain.
