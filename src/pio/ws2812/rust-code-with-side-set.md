# WS2812 Control on RP2040 Using PIO Side-Set in Rust

In this section, we implement the WS2812 control logic again, but this time using the side-set feature of the PIO instruction set.

Side-set allows us to change the pin state at the same time an instruction executes. Instead of using a separate set pins instruction, we can attach the pin transition directly to another instruction, reducing the total number of instructions in the loop.

The overall timing model remains the same. Each bit still occupies a fixed number of instruction cycles. The difference here is only in how the pin transitions are expressed inside the PIO program.

## PIO Program with Side-Set

Now let us look at the PIO program that uses side-set to generate the WS2812 waveform.

```armasm
.side_set 1

set pindirs, 1  side 0                 ; Configure the pin as output

.wrap_target
    bit_loop:
        out x, 1        side 0 [3]         ; Shift one bit into X. Keep the pin LOW.
        jmp !x do_zero  side 1 [2]         ; Branch on the bit. Drive the pin HIGH.
    do_one:
        jmp bit_loop    side 1 [2]         ; If the bit is 1, keep the pin HIGH longer.
    do_zero:
        nop             side 0 [2]         ; If the bit is 0, drive the pin LOW.
.wrap
```

`.side_set 1` declares that one side-set bit is available. This lets us control the output pin state together with each instruction. That means every instruction must specify a side 0 or side 1 value. That bit directly controls the configured side-set pin during the instruction cycle, including delay cycles.

`set pindirs, 1 side 0` configures the pin as output. At the same time, side 0 ensures the pin drives LOW.

## Cycle-by-Cycle Timing Analysis

We are still running the state machine at 8 MHz.

One instruction cycle = 125 ns
One bit period = 10 cycles = 1250 ns

We now verify that both logic paths consume exactly 10 cycles and that the pulse widths fall within the WS2812 timing window.

### Common Start for Both 0 and 1

Every bit begins at bit_loop.

```armasm
out x, 1        side 0 [3]
```

This instruction takes 1 execution cycle plus 3 delay cycles, giving a total of 4 cycles. During all 4 cycles, the pin is driven LOW because of side 0.

Next:

```armasm
jmp !x do_zero  side 1 [2]
```

This instruction consumes 1 execution cycle plus 2 delay cycles, for a total of 3 cycles. During all 3 cycles, the pin is driven HIGH because of side 1.

So before branching completes, we have consumed:

```
LOW = 4 cycles
HIGH = 3 cycles
Total = 7 cycles
```

At this point, 7 of the 10 cycles in the bit period have elapsed. Execution now diverges depending on whether the shifted bit was 0 or 1.

### Logic 1 Timing

If the shifted bit is 1, the jump is not taken and execution continues to do_one:

```armasm
jmp bit_loop    side 1 [2]
```

This instruction consumes 1 execution cycle plus 2 delay cycles, giving 3 cycles total. During all 3 cycles, the pin remains HIGH.

The full timing for a logic 1 bit therefore becomes:

```
LOW = 4 cycles
HIGH = 3 + 3 = 6 cycles
Total = 10 cycles
```

## Logic 0 Timing

If the shifted bit is 0, the branch jumps to do_zero:

```armasm
nop             side 0 [2]
```

This instruction consumes 1 execution cycle plus 2 delay cycles, giving 3 cycles total. During these 3 cycles, the pin is driven LOW.


The full timing for a logic 0 bit becomes:

```
LOW = 4 + 3 = 7 cycles
HIGH = 3 cycles
Total = 10 cycles
```

Both logic paths consume exactly 10 cycles.

## Binding the Side-Set Pin

On the Rust side, almost everything remains the same. The clock divider, FIFO configuration, shift behavior, and main transmission loop do not change.

The only difference is how the side-set pin is bound when loading the PIO program. We must provide one GPIO pin to serve as the side-set output. This is done when calling use_program:

```rust
cfg.use_program(&common.load_program(&prg.program), &[&out_pin]);
```

The second argument assigns the side-set pin required by `.side_set 1` in the PIO program.

## Clone the existing project

You can clone (or refer) project I created and navigate to the `using-sideset` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/pio/ws2812/using-sideset
```

## The Full Code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_rp::clocks::clk_sys_freq;
use embassy_time::Timer;

// defmt Logging
use defmt::info;
use defmt_rtt as _;

use panic_probe as _;

use embassy_rp::bind_interrupts;
use embassy_rp::peripherals::PIO0;
use embassy_rp::pio::program::pio_asm;
use embassy_rp::pio::{Config, FifoJoin, InterruptHandler, Pio, ShiftConfig, ShiftDirection};

use fixed::types::U24F8;

bind_interrupts!(struct Irqs {
    PIO0_IRQ_0 => InterruptHandler<PIO0>;
});

const CYCLES_PER_BIT: u32 = 10;

const NUM_LEDS: usize = 12;

const fn pack_grb(r: u8, g: u8, b: u8) -> u32 {
    ((g as u32) << 24) | ((r as u32) << 16) | ((b as u32) << 8)
}

const COLORS: [u32; 12] = [
    pack_grb(255, 0, 0),     // Red
    pack_grb(0, 255, 0),     // Green
    pack_grb(0, 0, 255),     // Blue
    pack_grb(255, 255, 0),   // Yellow
    pack_grb(255, 0, 255),   // Magenta
    pack_grb(0, 255, 255),   // Cyan
    pack_grb(255, 128, 0),   // Orange
    pack_grb(128, 0, 255),   // Purple
    pack_grb(255, 255, 255), // White
    pack_grb(255, 20, 147),  // Pink
    pack_grb(0, 255, 128),   // Spring Green
    pack_grb(255, 215, 0),   // Gold
];

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());
    info!("Initializing the program");

    let pio = p.PIO0;
    let Pio {
        mut common,
        mut sm0,
        ..
    } = Pio::new(pio, Irqs);

    let out_pin = common.make_pio_pin(p.PIN_15);

    let prg = pio_asm!(
        "
        .side_set 1

        set pindirs, 1  side 0                 ; Configure the pin as output

        .wrap_target
            bit_loop:
                out x, 1        side 0 [3]         ; Shift one bit into X. Keep the pin LOW.
                jmp !x do_zero  side 1 [2]         ; Branch on the bit. Drive the pin HIGH.
            do_one:
                jmp bit_loop    side 1 [2]         ; If the bit is 1, keep the pin HIGH longer.
            do_zero:
                nop             side 0 [2]         ; If the bit is 0, drive the pin LOW.
        .wrap
        "
    );

    let mut cfg = Config::default();
    cfg.use_program(&common.load_program(&prg.program), &[&out_pin]);
    cfg.set_set_pins(&[&out_pin]);

    let ws2812_freq = U24F8::from_num(800);
    let bit_freq = ws2812_freq * CYCLES_PER_BIT;

    let clock_freq = U24F8::from_num(clk_sys_freq() / 1000);
    cfg.clock_divider = clock_freq / bit_freq;

    // FIFO config
    cfg.fifo_join = FifoJoin::TxOnly;

    cfg.shift_out = ShiftConfig {
        auto_fill: true,
        threshold: 24,
        direction: ShiftDirection::Left,
    };

    sm0.set_config(&cfg);

    sm0.set_enable(true);

    loop {
        for i in 0..NUM_LEDS {
            sm0.tx().wait_push(COLORS[i]).await;
        }

        Timer::after_millis(100).await;
    }
}
```
