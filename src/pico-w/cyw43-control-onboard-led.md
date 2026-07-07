{{#title Initializing the Wi-Fi Controller and Controlling the Onboard LED}}

# Initializing the Wi-Fi Controller and Controlling the Onboard LED

At this point, the CYW43 driver is running in the background and is ready to communicate with the wireless chip. We can now use the `control` object returned by the driver to initialize the Wi-Fi controller and control the onboard LED.

The first step is to initialize the Wi-Fi controller.

```rust
control.init(clm).await;
```

Earlier, we learned that the CYW43439 requires two firmware files. The main firmware was transferred to the wireless chip when the driver was created. This call loads the Country Locale Matrix (CLM), which contains the country-specific regulatory information required for Wi-Fi operation.

Next, we configure the power management mode.

```rust
control
    .set_power_management(cyw43::PowerManagementMode::PowerSave)
    .await;
```

The CYW43439 supports several power management modes. In this example, we select `PowerSave`, which reduces power consumption when the wireless chip is idle.

Finally, we can control the onboard LED.

```rust
let delay = Duration::from_secs(1);

loop {
    info!("led on!");
    control.gpio_set(0, true).await;
    Timer::after(delay).await;

    info!("led off!");
    control.gpio_set(0, false).await;
    Timer::after(delay).await;
}
```

The call to `gpio_set()` changes the state of a GPIO on the CYW43439. On the Pico W, GPIO 0 of the CYW43439 is connected to the onboard LED, so passing `true` turns the LED on and passing `false` turns it off.
