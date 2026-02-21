# Using Embassy and Smart LEDs crate

So far, we implemented the WS2812 driver manually. We wrote the PIO program ourselves, calculated the timing in instruction cycles, configured the clock divider, and pushed raw 24-bit values into the TX FIFO.

In this section, we take a different approach. Instead of managing the PIO program and FIFO directly, we switch to Embassy's built-in WS2812 driver provided by embassy_rp. This driver already contains a validated PIO implementation and handles the low-level details internally.

On the color side, instead of manually packing GRB values into a 32-bit integer, we use the `RGB8` type from the smart_leds crate. This simply gives us a structured way to represent red, green, and blue components without dealing with bit shifts and masks.

The WS2812 protocol itself does not change. What changes is the abstraction level. We move from a fully manual PIO implementation to using a ready-made driver that handles the hardware details for us.


## Project from template

Generate a new project using the custom Embassy template.

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

### Additional Crates required

Add the following dependency to Cargo.toml along with the existing ones:

```toml
smart-leds = { version = "0.4.0" }
```

The smart-leds crate is used to represent LED color data in a structured way using types like `RGB8`, and it defines traits such as SmartLedsWrite and SmartLedsWriteAsync that LED driver crates implement to accept sequences of pixel values; it is protocol agnostic and does not implement any specific LED communication standard like WS2812 itself, which means the same color types can be used with different addressable LED drivers such as WS2812, SK6812, APA102, and others, while the actual signal timing and hardware control are handled by the respective driver.

## Additional Imports

We will import the required Embassy PIO modules, the built-in WS2812 program and driver, and the RGB8 type from the smart_leds crate.

```rust
use embassy_rp::bind_interrupts;

use smart_leds::RGB8;

use embassy_rp::peripherals::PIO0;
use embassy_rp::pio::{InterruptHandler, Pio};
use embassy_rp::pio_programs::ws2812::{PioWs2812, PioWs2812Program};
```


## Defining the Color Array

Previously, we defined the colors as packed 24-bit GRB values stored in a u32. Now, instead of manually shifting and combining bits, we define the colors directly using the RGB8 type from the smart_leds crate.

```rust
#[rustfmt::skip]
const COLORS: [RGB8; 12] = [
    RGB8 { r: 255, g: 0,   b: 0   }, // Red
    RGB8 { r: 255, g: 127, b: 0   }, // Orange
    RGB8 { r: 255, g: 255, b: 0   }, // Yellow
    RGB8 { r: 127, g: 255, b: 0   }, // Chartreuse
    RGB8 { r: 0,   g: 255, b: 0   }, // Green
    RGB8 { r: 0,   g: 255, b: 127 }, // Spring Green
    RGB8 { r: 0,   g: 255, b: 255 }, // Cyan
    RGB8 { r: 0,   g: 127, b: 255 }, // Azure
    RGB8 { r: 0,   g: 0,   b: 255 }, // Blue
    RGB8 { r: 127, g: 0,   b: 255 }, // Violet
    RGB8 { r: 255, g: 0,   b: 255 }, // Magenta
    RGB8 { r: 255, g: 0,   b: 127 }, // Rose
];
```

## Initializing the Embassy WS2812 Driver

The initialization of PIO is the same as before. Once we initialize the PIO block and obtain common and sm0, we use Embassy’s built-in WS2812 program and driver instead of loading our own PIO assembly.

```rust
let program = PioWs2812Program::new(&mut common);
let mut ws2812 = PioWs2812::new(&mut common, sm0, p.DMA_CH0, p.PIN_15, &program);
```

PioWs2812Program::new prepares the prebuilt WS2812 PIO program provided by Embassy.

PioWs2812::new attaches that program to the selected state machine, configures the output pin, and sets up DMA for transferring LED data. From this point onward, the driver handles the low-level PIO setup and transmission internally.

## Writing Color Data to the LEDs

```rust
ws2812.write(&COLORS).await;
```

Once the driver is initialized, sending data becomes a single call. We pass a reference to the COLORS array, and the driver handles transferring the color data to the LEDs.

Unlike the earlier manual implementation where we pushed packed values into the FIFO ourselves, this call delegates the entire transmission process to the Embassy driver.

## Clone the existing project

You can clone (or refer) project I created and navigate to the `using-embassy` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/pio/ws2812/using-embassy
```

## The Full Code

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

use smart_leds::RGB8;

use embassy_rp::peripherals::PIO0;
use embassy_rp::pio::{InterruptHandler, Pio};
use embassy_rp::pio_programs::ws2812::{PioWs2812, PioWs2812Program};

bind_interrupts!(struct Irqs {
    PIO0_IRQ_0 => InterruptHandler<PIO0>;
});

#[rustfmt::skip]
const COLORS: [RGB8; 12] = [
    RGB8 { r: 255, g: 0,   b: 0   }, // Red
    RGB8 { r: 255, g: 127, b: 0   }, // Orange
    RGB8 { r: 255, g: 255, b: 0   }, // Yellow
    RGB8 { r: 127, g: 255, b: 0   }, // Chartreuse
    RGB8 { r: 0,   g: 255, b: 0   }, // Green
    RGB8 { r: 0,   g: 255, b: 127 }, // Spring Green
    RGB8 { r: 0,   g: 255, b: 255 }, // Cyan
    RGB8 { r: 0,   g: 127, b: 255 }, // Azure
    RGB8 { r: 0,   g: 0,   b: 255 }, // Blue
    RGB8 { r: 127, g: 0,   b: 255 }, // Violet
    RGB8 { r: 255, g: 0,   b: 255 }, // Magenta
    RGB8 { r: 255, g: 0,   b: 127 }, // Rose
];

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    info!("Initializing the program");

    let Pio {
        mut common, sm0, ..
    } = Pio::new(p.PIO0, Irqs);

    let program = PioWs2812Program::new(&mut common);
    let mut ws2812 = PioWs2812::new(&mut common, sm0, p.DMA_CH0, p.PIN_15, &program);

    ws2812.write(&COLORS).await;

    loop {
        Timer::after_millis(100).await;
    }
}
```
