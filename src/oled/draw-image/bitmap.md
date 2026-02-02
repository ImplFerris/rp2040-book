# Using Bitmap Image file

You can use BMP (.bmp) files directly instead of raw image data by utilizing the [tinybmp](https://docs.rs/tinybmp/latest/tinybmp/) crate. tinybmp is a lightweight BMP parser designed for embedded environments. While it is mainly intended for drawing BMP images to embedded_graphics DrawTargets, it can also be used to parse BMP files for other applications.

## BMP file

The crate requires the image to be in BMP format. If your image is in another format, you will need to convert it to BMP. For example, you can use the following command on Linux to convert a PNG image to a monochrome BMP:

```sh
convert ferris.png -monochrome ferris.bmp
```

I have created the Ferris BMP file, which you can use for this exercise. Download it from [here](../images/ferris.bmp).

<img style="display: block; margin: auto;" alt="ferris bmp file" src="../images/ferris.bmp"/>


## Project base

We will copy the oled-rawimg project and work on top of that. 

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cp -r rp2040-projects/embassy/oled/oled-rawimg ~/YOUR_PROJECT_FOLDER/oled-bmp
```

or you can simply create a fresh project from the template and follow the same steps we used earlier.

## Update Cargo.toml

We need one more crate called "tinybmp" to load the bmp image.

```toml
tinybmp = "0.6.0"
```

## Using the BMP File

Place the "ferris.bmp" file inside the src folder. The code is pretty straightforward: load the image as bytes and pass it to the from_slice function of the Bmp. Then, you can use it with the Image.

```rust
// the usual boilerplate code goes here...

// Include the BMP file data.
let bmp_data = include_bytes!("../ferris.bmp");

// Parse the BMP file.
let bmp = Bmp::from_slice(bmp_data).unwrap();

// usual code:
let image = Image::new(&bmp, Point::new(32, 0));

image
    .draw(&mut display)
    .expect("failed to draw text to display");

defmt::info!("Displaying image");
display.flush().await.expect("failed to flush data to display");
```

## Clone the existing project

You can also clone (or refer) project I created and navigate to the `ferris-image` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/oled/ferris-image
```

## Full code

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
use tinybmp::Bmp;

bind_interrupts!(struct Irqs {
    I2C0_IRQ => i2c::InterruptHandler<I2C0>;
});

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

    // Include the BMP file data.
    let bmp_data = include_bytes!("../ferris.bmp");

    // Parse the BMP file.
    let bmp = Bmp::from_slice(bmp_data).unwrap();

    // usual code:
    let image = Image::new(&bmp, Point::new(32, 0));

    image
        .draw(&mut display)
        .expect("failed to draw text to display");

    defmt::info!("Displaying image");
    display
        .flush()
        .await
        .expect("failed to flush data to display");

    loop {
        Timer::after_secs(1).await;
    }
}
``` 
