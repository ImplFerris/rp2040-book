# Looping PIO Programs Using .wrap and .wrap_target directive

In the previous chapter, we looped our PIO program using the `jmp` instruction. That approach works well and makes control flow obvious.

PIO also provides another way to loop programs automatically, using `.wrap` and `.wrap_target`. These are not instructions. They are directives that control how the assembler sets up program flow.

> [!Note]
> A directive is an instruction to the assembler, not to the PIO hardware. It does not execute at runtime and does not consume any clock cycles. It only tells the assembler how to arrange the program.

## The .wrap_target Directive

The `.wrap_target` directive marks the instruction where execution should continue after wrapping.

It must be placed immediately before an instruction. That instruction becomes the wrap target.

Rules for .wrap_target:

- It can only be used inside a PIO program.
- It may only appear once.
- If it is not specified, the wrap target defaults to the first instruction of the program.


## The .wrap Directive

The `.wrap` directive marks the point where wrapping occurs.

It must be placed immediately after an instruction. When that instruction finishes and execution reaches this point through normal control flow, the program counter wraps back to the instruction marked by ".wrap_target".

Wrapping only happens on normal fall-through execution. If a `jmp` instruction transfers control elsewhere, wrapping does not occur.

Rules for .wrap:

- It can only be used inside a PIO program.
- It may only appear once.
- If .wrap is not specified, the program automatically wraps after the final instruction and execution continues from the wrap target.

## Rewriting the Blink Program Using Wrap Directives

Now we rewrite our LED blink program using wrapping instead of an explicit `jmp`.

```rust
set pindirs, 1

.wrap_target
    set pins, 1 [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    set pins, 0 [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
    nop [31]
.wrap
```

There is no `jmp` instruction here. Execution flows linearly through the program. When the instruction before ".wrap" completes and control reaches this point normally, the program counter automatically wraps back to the instruction marked by ".wrap_target".

## Timing Differences Compared to Using JMP

When we used `jmp loop`, that jump instruction consumed one clock cycle. During that cycle, the pin remained in its previous state. This is why we previously had to adjust delay values to keep the duty cycle balanced.

With wrapping, no extra instruction cycle is consumed. Because of this, both `set pins, 1` and `set pins, 0` can use the same delay value without affecting the timing balance.
