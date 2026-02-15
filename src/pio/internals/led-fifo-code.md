# Using PIO FIFO on Raspberry Pi Pico to Send Bit Sequences

Generate a new project using the custom Embassy template.

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

I will avoid repeating the steps we used earlier to initialize the PIO block and the state machine. Those steps are exactly the same as in the LED blinking example. Here, we will focus only on the PIO program and the parts that are directly related to the FIFO.

## Defining the PIO Program

We will write a simple PIO program that keeps shifting bits to the GPIO pin in a loop. 

```rust
let prg = pio_asm!(
    "
    set pindirs, 1
    .wrap_target
        out pins,1 [31]
    .wrap
    "
    );
```
The program itself will only execute `out pins, 1` repeatedly. New 32-bit values will come from the FIFO automatically when the OSR becomes empty since we enabled automatic refill in the configuration.  Each time this instruction runs, one bit is shifted from the output shift register (OSR) to the GPIO pin.

## Configuring the OUT Pins and Automatic Refill

We will configure the state machine so that the `out` instruction drives our chosen GPIO pin.

```rust
cfg.set_out_pins(&[&out_pin]);
cfg.shift_out.auto_fill = true;
```

The important configuration here is `auto_fill`. When enabled, the hardware automatically refills the OSR from the TX FIFO once all 32 bits have been shifted out. This means we do not need to write a `pull` instruction in the PIO program. The FIFO and the OSR work together automatically.


## Preparing the Bit Sequences

Each element in this array is a 32-bit value. These are structured bit sequences composed of 1s and 0s. When we shift these words out one bit at a time, the LED turns ON when a `1` is shifted out and OFF when a `0` is shifted out. Even though the transitions happen quickly, the overall pattern produces visible differences in brightness and behavior.

```rust
let patterns = [
    0b10000000_10000000_10000000_10000000,
    0b11000000_11000000_11000000_11000000,
    0b11110000_11110000_11110000_11110000,
    0b11111100_11111100_11111100_11111100,
    0b00000000_00000000_00000000_00000000,
];
```

You could write these values in normal decimal or hexadecimal form, but I am using binary here so the pattern of 1s and 0s is clearly visible.

## Sending Data Through the FIFO

Inside this loop, we will push each 32-bit value into the TX FIFO using `wait_push`. The state machine continues running independently. As it shifts out all 32 bits from the OSR, the hardware automatically refills it from the FIFO when more data is available.

```rust
 loop {
    for pattern in &patterns {
        sm0.tx().wait_push(*pattern).await;
        info!("Pushed pattern to FIFO {:032b}", pattern);
        Timer::after_millis(100).await;
    }

    Timer::after_secs(3).await;
}
```

The FIFO acts as a buffer between the CPU and the PIO state machine. The CPU produces data. The PIO consumes it at its own pace. Once this flow is clear, we can build on it for more advanced protocols and devices.


## Clone the existing project

You can clone (or refer) project I created and navigate to the `hello-fifo` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/pio/hello-fifo/
```

## The Full code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;

// defmt Logging
use defmt::info;
use defmt_rtt as _;

use panic_probe as _;

use embassy_rp::bind_interrupts;
use embassy_rp::peripherals::PIO0;
use embassy_rp::pio::program::pio_asm;
use embassy_rp::pio::{Config, InterruptHandler, Pio};

bind_interrupts!(struct Irqs {
    PIO0_IRQ_0 => InterruptHandler<PIO0>;
});

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

    let prg = pio_asm!(
        "
    set pindirs, 1
    .wrap_target
        out pins,1 [31]
    .wrap
    "
    );

    let out_pin = common.make_pio_pin(p.PIN_15);

    let mut cfg = Config::default();
    cfg.use_program(&common.load_program(&prg.program), &[]);
    cfg.set_set_pins(&[&out_pin]);
    cfg.clock_divider = 65535u16.into();

    cfg.set_out_pins(&[&out_pin]);
    cfg.shift_out.auto_fill = true;

    sm0.set_config(&cfg);

    sm0.set_enable(true);

    let patterns = [
        0b10000000_10000000_10000000_10000000,
        0b11000000_11000000_11000000_11000000,
        0b11110000_11110000_11110000_11110000,
        0b11111100_11111100_11111100_11111100,
        0b00000000_00000000_00000000_00000000,
    ];

    loop {
        for pattern in &patterns {
            sm0.tx().wait_push(*pattern).await;
            info!("Pushed pattern to FIFO {:032b}", pattern);
            Timer::after_millis(100).await;
        }

        Timer::after_secs(3).await;
    }
}
```
