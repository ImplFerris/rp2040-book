# LED Dimming on Raspberry Pi Pico with Embassy

Let's create a dimming LED effect using PWM on the Raspberry Pi Pico with Embassy.

## Project from template

Generate a new project using the custom Embassy template.

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

## Update Imports

Add the import below to bring the PWM types into scope:

```rust
use embassy_rp::pwm::{Pwm, SetDutyCycle};
```

## Initialize PWM

Let's set up the PWM for the LED. Use the first line for an external LED on GPIO 15, or uncomment the second one if you want to use the onboard LED.

```rust
// For external LED on GPIO 15
let mut pwm = Pwm::new_output_b(p.PWM_SLICE7, p.PIN_15, Default::default());

// For onboard LED
// If you are using Pico W, follow the onboard LED steps described in the quick start section.
// let mut pwm = Pwm::new_output_a(p.PWM_SLICE0, p.PIN_15, Default::default());
```

## Main logic

In the main loop, we create the fade effect by increasing the duty cycle from 0 to 100 percent and then bringing it back down. The small delay between each step makes the dimming smooth. You can adjust the delay and observe how the fade speed changes.

```rust
loop {
    for i in 0..=100 {
        Timer::after_millis(8).await;
        let _ = pwm.set_duty_cycle_percent(i);
    }
    
    for i in (0..=100).rev() {
        Timer::after_millis(8).await;
        let _ = pwm.set_duty_cycle_percent(i);
    }

    Timer::after_millis(500).await;
}
```

## The full code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;

// defmt Logging
use defmt::info;
use defmt_rtt as _;

use embassy_rp::pwm::{Pwm, SetDutyCycle};
use panic_probe as _;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    // For external LED on GPIO 15
    let mut pwm = Pwm::new_output_b(p.PWM_SLICE7, p.PIN_15, Default::default());

    // For onboard LED
    // If you are using Pico W, follow the onboard LED steps described in the quick start section.
    // let mut pwm = Pwm::new_output_a(p.PWM_SLICE0, p.PIN_15, Default::default());

    info!("Initializing the program");

    loop {
        for i in 0..=100 {
            Timer::after_millis(8).await;
            let _ = pwm.set_duty_cycle_percent(i);
        }

        for i in (0..=100).rev() {
            Timer::after_millis(8).await;
            let _ = pwm.set_duty_cycle_percent(i);
        }

        Timer::after_millis(500).await;
    }
}
```


## Clone the existing project

You can clone the project I created and navigate to the `led-dimming` folder:

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/led-dimming
```
