# Mapping the Bit Period into Instruction Cycles

The WS2812 protocol defines timing in nanoseconds. In a PIO program, however, we cannot directly specify a delay like "wait 350 ns." Timing is controlled through instruction cycles.

Each PIO instruction takes exactly one state machine clock cycle, and the actual duration of that cycle depends on the clock frequency we configure. By selecting an appropriate clock frequency and arranging how many cycles each operation consumes, we can generate the required HIGH and LOW pulse widths.

The next step is to determine how many cycles correspond to one bit period (1.25 µs), and then decide how many of those cycles the signal should remain HIGH. Once this mapping is defined, the PIO program can be written so that each bit consumes the correct number of cycles.

## Deciding the State Machine Clock Frequency

The WS2812 bit time is 1250 ns (1.25 µs). If we ran the PIO at 800 kHz, one instruction cycle would also take 1250 ns.

As we saw earlier, a single bit consists of both a HIGH pulse and a LOW pulse. The only difference between 0 and 1 is how long the signal stays HIGH within that 1250 ns window. If one instruction already takes the entire 1250 ns, then a single instruction would consume the whole bit period. There would be no way to generate separate HIGH and LOW phases, and no way to create the required 350 ns or 700 ns HIGH pulses.

> [!Important]
> The instruction cycle must be much shorter than 1250 ns.

If we instead run the state machine at 8 MHz, one cycle becomes 125 ns. Now the 1250 ns bit period consists of 10 instruction cycles.

## Timing to Cycles

As we already learnt in the previous chapter, this is approximate time required for informing the WS2812 LED whether the bit is 0 or 1.

- Logic 0: HIGH for about 350 ns, then LOW for about 800 ns
- Logic 1: HIGH for about 700 ns, then LOW for about 600 ns

Since one instruction cycle is 125 ns at 8 MHz, we can now convert these timings into cycles.

The total bit time is:

```sh
1250 ns / 125 ns = 10 cycles per bit
```

So each bit must consume 10 PIO cycles in total.

<div class="image-with-caption" style="text-align:center;">
    <img src="./images/WS2812 Raspberry Pi Pico Clock Cycle Timing at 8MHz.svg" alt="WS2812 Raspberry Pi Pico Clock Cycle Timing at 8MHz" style="height:auto; display:block; margin:auto;"/>
</div>

For a logic 0 bit:

```sh
HIGH ≈ 350 ns  →  350 / 125 ≈ 3 cycles
LOW  ≈ 800 ns  →  remaining 7 cycles
```

For a logic 1 bit:

```sh
HIGH ≈ 700 ns  →  700 / 125 ≈ 6 cycles
LOW  ≈ 600 ns  →  remaining 4 cycles
```

The values are approximate. The WS2812 protocol allows timing tolerance (±150 ns), so rounding to the nearest whole cycle remains within specification.

## Deriving the PIO Clock Divider

We already determined that the state machine must run at 8 MHz so that one WS2812 bit occupies 10 instruction cycles. Now we derive the clock divider required to generate that 8 MHz state machine frequency from the system clock.

On the RP2040, the default system clock is 125 MHz. The relationship between them is:

```sh
state_machine_frequency = clk_sys / divider
```

Rearranging:
```sh
divider = clk_sys / state_machine_frequency
```

Substituting the values:
```sh
divider = 125 MHz / 8 MHz = 15.625
```

Instead of calculating this manually, we compute it directly in code:

```rust
const CYCLES_PER_BIT: u32 = 10;

let ws2812_freq = U24F8::from_num(800);
let bit_freq = ws2812_freq * CYCLES_PER_BIT;

let clock_freq = U24F8::from_num(clk_sys_freq() / 1000);
cfg.clock_divider = clock_freq / bit_freq;
```

ws2812_freq represents the 800 kHz bit rate. Multiplying it by CYCLES_PER_BIT produces the required instruction frequency, which is 8000 kHz, or 8 MHz.

clk_sys_freq() returns the system clock in hertz, so dividing by 1000 converts it to kilohertz to match units.

Finally, dividing the system clock by the required instruction frequency gives the clock divider. With a 125 MHz system clock, the result is 15.625.
