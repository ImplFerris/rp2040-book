{{#title Creating and Running the CYW43 Driver}}

# Creating and Running the CYW43 Driver

We now have everything needed to create the CYW43 driver. We have already created the PIO-based SPI interface that communicates with the CYW43439, and the firmware is ready to be uploaded to the wireless chip.

The first step is to create the driver's state, which stores the internal state required by the driver.

```rust
static STATE: StaticCell<cyw43::State> = StaticCell::new();
let state = STATE.init(cyw43::State::new());
```

The driver state must remain available for the lifetime of the program. We therefore store it inside a `StaticCell`, which provides a statically allocated location for the state.

A `StaticCell` starts out empty. Calling init function stores the `State` inside the cell and returns a mutable reference to it. Because the value is stored in statically allocated memory, this reference remains valid for the lifetime of the program and can safely be passed to the CYW43 driver.

> [!TIP]
> You might wonder why we don't simply write `let state = &mut cyw43::State::new();`. The `state` is shared by the objects returned from `cyw43::new()`, including the `runner`, which must keep running in the background for the lifetime of the application. Because of this, the driver state must also remain available for the entire lifetime of the program. `StaticCell` provides a statically allocated location where it can safely live.

With the driver state ready, we can create the CYW43 driver.

```rust
let (_net_device, mut control, runner) = cyw43::new(state, pwr, spi, fw).await;
```

This function creates the CYW43 driver using the driver state, the power control pin, the PIO-based SPI interface, and the firmware image. During initialization, the driver transfers the firmware to the CYW43439 so that the wireless chip can begin executing it.

The function returns three values.

The first value, `_net_device`, represents the network device that is later used by the networking stack. Since this example only controls the onboard LED, we do not use it, so we simply prefix the variable name with an underscore.

The second value, `control`, provides an interface for controlling the CYW43439. We will use it to initialize the Wi-Fi controller, configure power management, and control the onboard LED.

The third value, `runner`, is responsible for handling communication with the CYW43439 in the background.

## Running the Driver

Creating the driver is only the first step. For the driver to communicate with the CYW43439, the `runner` must continue running for as long as the application uses the wireless chip.

We first define a background Embassy task that owns the runner.

```rust
#[embassy_executor::task]
async fn cyw43_task(
    runner: cyw43::Runner<'static, Output<'static>, PioSpi<'static, PIO0, 0, DMA_CH0>>,
) -> ! {
    runner.run().await
}
```

The task itself is simple, just calls `runner.run().await`. This function never returns. Instead, it runs continuously in the background, handling communication between the RP2040 and the CYW43439.

Next, we spawn this task.

```rust
unwrap!(spawner.spawn(cyw43_task(runner)));
```

This starts the background task. While it continuously manages communication with the CYW43439, the `main` task is free to continue executing the rest of our program.

In the next section, we will use the `control` object returned by the driver to initialize the Wi-Fi controller and control the onboard LED.
