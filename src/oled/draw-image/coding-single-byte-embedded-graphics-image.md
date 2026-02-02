## Drawing a Single Byte Image in Embedded Rust using embedded-graphics

By now, i hope you understand how the image is represented in the byte array. Now, let's move on to the coding part.


## Project from template

Generate a new project using the custom Embassy template.

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

## Update Dependencies

Add the following lines to your Cargo.toml under dependencies:

```toml
embedded-graphics = "0.8.1"
ssd1306 = { version = "0.10.0", features = ["async"] }
```

## Additional imports

Add these imports at the top of your main.rs:

```rust
// Interrupt Binding
use embassy_rp::peripherals::I2C0;
use embassy_rp::{bind_interrupts, i2c};

// I2C
use embassy_rp::i2c::{Config as I2cConfig, I2c};

// OLED
use ssd1306::{I2CDisplayInterface, Ssd1306Async, prelude::*};

// Embedded Graphics
use embedded_graphics::{
    image::{Image, ImageRaw},
    pixelcolor::BinaryColor,
    prelude::Point,
    prelude::*,
};
```

## Boilerplate codes

We have already explained this part in the previous chapter.

### Bind I2C Interrupt

```rust
bind_interrupts!(struct Irqs {
    I2C0_IRQ => i2c::InterruptHandler<I2C0>;
});
```

### Initialize I2C and Display instance

```rust
let sda = p.PIN_16;
let scl = p.PIN_17;

let mut i2c_config = I2cConfig::default();
i2c_config.frequency = 400_000; // 400kHz

let i2c_bus = I2c::new_async(p.I2C0, scl, sda, Irqs, i2c_config);

let i2c_interface = I2CDisplayInterface::new(i2c_bus);

let mut display = Ssd1306Async::new(i2c_interface, DisplaySize128x64, DisplayRotation::Rotate0)
    .into_buffered_graphics_mode();

display
    .init()
    .await
    .expect("failed to initialize the display");
```

## Create Your Image

We store our image as a byte array. Each byte represents one row of pixels.

```rust
// 8x5 pixels
#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    0b00111000,
    0b01000100,
    0b01000100,
    0b00101000,
    0b11101110,
];
```

This creates an Ohm symbol (Ω) that's 8 pixels wide and 5 pixels tall. Each 0b means we're writing in binary using 1s and 0s. A 1 means the pixel is ON (white), and a 0 means the pixel is OFF (black). Each line represents one row of the image from top to bottom.

## Draw the Image

Now let's put the image on the display:

```rust
let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 8);
let image = Image::new(&raw_image, Point::zero());
```

The first line creates a raw image from our byte data. We tell it the image is 8 pixels wide, and it figures out the height by itself. The second line places the image at position (0, 0), which is the top-left corner of the screen.

## Display the Image

Just like in the previous chapter, we need to draw the image to the display buffer and then send it to the screen:

```rust
image.draw(&mut display).expect("failed to draw text to display");
display.flush().await.expect("failed to flush data to display");
```

## Clone the existing project

You can also clone (or refer) project I created and navigate to the `byte-image` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/oled/byte-image
```

## The Complete Code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;

// defmt Logging
use defmt::info;
use defmt_rtt as _;

use panic_probe as _;

// Interrupt Binding
use embassy_rp::peripherals::I2C0;
use embassy_rp::{bind_interrupts, i2c};

// I2C
use embassy_rp::i2c::{Config as I2cConfig, I2c};

// OLED
use ssd1306::{I2CDisplayInterface, Ssd1306Async, prelude::*};

// Embedded Graphics
use embedded_graphics::{
    image::{Image, ImageRaw},
    pixelcolor::BinaryColor,
    prelude::Point,
    prelude::*,
};

bind_interrupts!(struct Irqs {
    I2C0_IRQ => i2c::InterruptHandler<I2C0>;
});

// 8x5 pixels
#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    0b00111000,
    0b01000100,
    0b01000100,
    0b00101000,
    0b11101110,
];

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    info!("Initializing the program");

    let sda = p.PIN_16;
    let scl = p.PIN_17;

    let mut i2c_config = I2cConfig::default();
    i2c_config.frequency = 400_000; // 400kHz

    let i2c_bus = I2c::new_async(p.I2C0, scl, sda, Irqs, i2c_config);

    let i2c_interface = I2CDisplayInterface::new(i2c_bus);

    let mut display = Ssd1306Async::new(i2c_interface, DisplaySize128x64, DisplayRotation::Rotate0)
        .into_buffered_graphics_mode();

    display
        .init()
        .await
        .expect("failed to initialize the display");

    let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 8);
    let image = Image::new(&raw_image, Point::zero());

    image
        .draw(&mut display)
        .expect("failed to draw text to display");
    display
        .flush()
        .await
        .expect("failed to flush data to display");

    loop {
        Timer::after_secs(1).await;
    }
}
```
