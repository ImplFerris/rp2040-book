# Blinking onboard LED on Raspberry Pi Pico

When you start with embedded programming, GPIO is the first peripheral you'll work with. "General-Purpose Input/Output" means exactly what it sounds like: we can use it for both input and output. As an output, the Pico can send signals to control components like LEDs. As an input, components like buttons can send signals to the Pico.

For this exercise, we'll control the onboard LED by sending signals to it. If you check page 7 of the [Pico datasheet](https://datasheets.raspberrypi.com/pico/pico-datasheet.pdf#page=8), you'll see that the onboard LED is wired to GPIO Pin 25.

We'll configure GPIO Pin 25 as an output pin and set its initial state to low (off):

> [!Important]
> If you are using the Pico W variant, the onboard LED is controlled by the Wi-Fi chip and requires additional setup. For simplicity, it is recommended to use an external LED connected to a GPIO pin when following this section.

```rust
let mut led = Output::new(peripherals.PIN_25, Level::Low);
// When using Pico W, use external LED connected on GPIO 13 or other GPIO Pin
// let mut led = Output::new(peripherals.PIN_13, Level::Low);
```

Most code editors like VS Code have shortcuts to automatically add imports for you. If your editor doesn't have this feature or you're having issues, you can manually add these imports:

```rust
use embassy_rp::gpio::{Level, Output};
```

## Blinking Logic

Now we'll create a simple loop to make the LED blink. First, we turn on the LED by calling the `set_high()` function on our GPIO instance. Then we add a short delay using Timer. Next, we turn off the LED with `set_low()`. Then we add another delay. This creates the blinking effect.

Let's import Timer into our project:

```rust
use embassy_time::Timer;
```

Here's the blinking loop:

```rust
loop {
    led.set_high();
    Timer::after_millis(250).await;

    led.set_low();
    Timer::after_millis(250).await;
}
```
