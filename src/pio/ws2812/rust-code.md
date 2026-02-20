# Code to Control WS2812 Using Raspberry Pi Pico's PIO in Embedded Rust

Let us move on to the Rust part. 

## Project

Set up the project as usual.  Then add the following dependency, which we will use to calculate and configure the clock divider value:

```toml
fixed = "1.30.0"
```

## Packing Color Data in GRB Format

Before sending data to the WS2812, we need to pack the red, green, and blue values into a 32-bit value in the format expected by the LED.

WS2812 does not use RGB order. It expects the data in GRB order, with the most significant bit transmitted first. Each LED consumes 24 bits in this order:

```
Green → Red → Blue
```

In the following function, we will pack the three 8-bit color values into the correct layout:

```rust
const fn pack_grb(r: u8, g: u8, b: u8) -> u32 {
    ((g as u32) << 24) | ((r as u32) << 16) | ((b as u32) << 8)
}
```

The left shift operator `<<` moves bits towards the most significant end. Here, each color byte starts as an 8-bit value, and we slide it into its target position within the 32-bit value:

```
g = 0xFF

As u32      :   00000000 00000000 00000000 11111111   
After << 24 :   11111111 00000000 00000000 00000000   (bits 31:24)

r = 0xFF

As u32      :   00000000 00000000 00000000 11111111   
After << 16 :   00000000 11111111 00000000 00000000   (bits 23:16)

b = 0xFF

As u32      :   00000000 00000000 00000000 11111111   
After << 8  :   00000000 00000000 11111111 00000000   (bits 15:8)
```

The three shifted values are then combined with bitwise OR `|`, merging them into a single 32-bit value since their bit ranges do not overlap:
```
  11111111 00000000 00000000 00000000 
| 00000000 11111111 00000000 00000000
| 00000000 00000000 11111111 00000000 
= 11111111 11111111 11111111 00000000   final value
```

This produces a 32-bit value arranged like this:

```
     ┌──────────┬──────────┬──────────┬──────────┐
     │  GREEN   │   RED    │   BLUE   │ (unused) │
     │  [31:24] │  [23:16] │  [15:8]  │  [7:0]   │
     └──────────┴──────────┴──────────┴──────────┘
         8 bits     8 bits     8 bits     8 bits
      ↑ MSB transmitted first
```

Green occupies the highest byte, followed by red, then blue. The lowest 8 bits are left as zero.


## Defining Color Data and Configuration

Next, we define the constants and color data that will be transmitted to the LEDs.

```rust
const CYCLES_PER_BIT: u32 = 10;

const NUM_LEDS: usize = 12;

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
```

CYCLES_PER_BIT must match the number of instruction cycles per bit that was used earlier when calculating the 8 MHz state machine frequency and the clock divider.

NUM_LEDS defines how many LEDs are connected in the chain. The one I am using has 12 LEDs. You can change this value as needed, but make sure to update the COLORS array so its length matches NUM_LEDS.

COLORS is a fixed array of 12 packed GRB values. The function takes RGB inputs but rearranges them into GRB format as required by the WS2812. Each entry represents the 24-bit color data that will be transmitted to one LED in sequence.


## Setting the Clock Divider

Let's set the state machine to run at 8 MHz so that each WS2812 bit occupies 10 instruction cycles.

```rust
let ws2812_freq = U24F8::from_num(800);
let bit_freq = ws2812_freq * CYCLES_PER_BIT;

let clock_freq = U24F8::from_num(clk_sys_freq() / 1000);
cfg.clock_divider = clock_freq / bit_freq;
```

## Configuring the FIFO

We have not used this configuration so far:

```rust
cfg.fifo_join = FifoJoin::TxOnly;
```

By default, each PIO state machine has separate TX and RX FIFOs, each with a depth of 4 entries. When we set FifoJoin::TxOnly, the TX and RX FIFOs are combined into a single TX FIFO with a depth of 8 entries.

In this project, we are only transmitting data to the WS2812. We do not use the RX FIFO at all. So we can join them and use the entire buffer for transmission. This gives us a larger TX FIFO, which allows more color data to be queued before the CPU needs to push additional values.


## Configuring Shift Behavior

This configuration controls how data is shifted out of the Output Shift Register (OSR).

```rust
cfg.shift_out = ShiftConfig {
    auto_fill: true,
    threshold: 24,
    direction: ShiftDirection::Left,
};
```

- direction: ShiftDirection::Left ensures that the most significant bit is transmitted first. 

- threshold: 24 tells the state machine to automatically pull new data after 24 bits have been shifted out. Since each WS2812 LED consumes exactly 24 bits (8 bits each for Green, Red, and Blue), this aligns the data stream correctly.

- auto_fill: true enables automatic pulling from the TX FIFO into the OSR when the threshold is reached. This allows continuous transmission without manually triggering a pull instruction.


## Main Program Loop

Inside the loop, we push one packed 24-bit color value at a time into the TX FIFO.

```rust
loop {
    for i in 0..NUM_LEDS {
        sm0.tx().wait_push(COLORS[i]).await;
    }

    Timer::after_millis(100).await;
}
```

`sm0.tx().wait_push(COLORS[i]).await` writes a single 32-bit value to the state machine. Since we configured auto pull with a threshold of 24 bits, the state machine automatically consumes 24 bits for each LED and then pulls the next value from the FIFO.

The for loop sends color data for all the LEDs in sequence. Because WS2812 LEDs are daisy chained, each LED consumes the first 24 bits meant for it and forwards the remaining bits to the next LED.

After transmitting the full frame, we wait for 100 milliseconds before sending the data again. The WS2812 requires a reset period of at least 50 microseconds to latch the data, and this delay is more than sufficient.

This continuously refreshes the LED chain with the same set of colors.

## Clone the existing project

You can clone (or refer) project I created and navigate to the `hello-ws2812` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/pio/ws2812/hello-ws2812
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


const fn pack_grb(r: u8, g: u8, b: u8) -> u32 {
    ((g as u32) << 24) | ((r as u32) << 16) | ((b as u32) << 8)
}

const CYCLES_PER_BIT: u32 = 10;

const NUM_LEDS: usize = 12;

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
        set pindirs, 1

        .wrap_target
        bit_loop:
            set pins, 0 [1]
            out x, 1
            jmp !x do_zero
        do_one:
            set pins, 1 [4]
            jmp bit_loop
        do_zero:
            set pins, 1 [2]
            set pins, 0 [2]
        .wrap
        "
    );

    let mut cfg = Config::default();
    cfg.use_program(&common.load_program(&prg.program), &[]);

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
