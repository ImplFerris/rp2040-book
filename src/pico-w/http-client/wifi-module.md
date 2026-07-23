# Understanding the Wi-Fi Module

Most of the code in `wifi.rs` should already look familiar. We covered the PIO SPI interface, the CYW43439 firmware, and the CYW43 driver in the previous chapters. Here, I have simply organized that code into a few helper functions to keep `main.rs` small and easier to follow.

## Initializing the CYW43 Driver

The first function called from `main.rs` is `init_cyw43`, so let's take a look at it. It takes the PIO peripheral, DMA channel, GPIO pins, and the task spawner as parameters.

Its implementation is almost identical to the code we wrote in the previous chapter. It loads the CYW43439 firmware, creates the PIO-based SPI interface, initializes the CYW43 driver, and starts the background task that keeps the driver running.

The only difference is that all of this setup has been moved into a helper function. Once the initialization is complete, the function returns the `net_device` and `control` handles, which we will use throughout the rest of the networking code.

```rust
pub async fn init_cyw43(
    spawner: Spawner,
    pio0: Peri<'static, PIO0>,
    dma_ch0: Peri<'static, DMA_CH0>,
    pin23: Peri<'static, embassy_rp::peripherals::PIN_23>,
    pin24: Peri<'static, embassy_rp::peripherals::PIN_24>,
    pin25: Peri<'static, embassy_rp::peripherals::PIN_25>,
    pin29: Peri<'static, embassy_rp::peripherals::PIN_29>,
) -> (cyw43::NetDriver<'static>, cyw43::Control<'static>) {
    // Load the CYW43439 firmware files
    let fw = cyw43::aligned_bytes!("../firmware/43439A0.bin");
    let clm = cyw43::aligned_bytes!("../firmware/43439A0_clm.bin");
    let nvram = cyw43::aligned_bytes!("../firmware/nvram_rp2040.bin");

    // Initialize the PIO SPI interface
    let pwr = Output::new(pin23, Level::Low);
    let cs = Output::new(pin25, Level::High);
    let mut pio = Pio::new(pio0, Irqs);
    let spi = PioSpi::new(
        &mut pio.common,
        pio.sm0,
        DEFAULT_CLOCK_DIVIDER,
        pio.irq0,
        cs,
        pin24,
        pin29,
        dma::Channel::new(dma_ch0, Irqs),
    );

    // Initialize the driver state
    static STATE: StaticCell<cyw43::State> = StaticCell::new();
    let state = STATE.init(cyw43::State::new());

    let (net_device, mut control, runner) = cyw43::new(state, pwr, spi, fw, nvram).await;

    spawner.spawn(unwrap!(cyw43_task(runner)));
    

    control.init(clm).await;
    control
        .set_power_management(cyw43::PowerManagementMode::PowerSave)
        .await;

    (net_device, control)
}
```

## Creating the Network Stack

The next step is to create the networking stack. This is the first new component that we have not seen before.  The `init_net_stack` function takes the task `Spawner`, the `net_device` returned by the CYW43 driver, and a random seed.

The networking stack also needs an IP configuration. In this example, we use DHCP, which allows your router to automatically assign an IP address to the Pico W. If you prefer to use a static IP address, you can uncomment the static IP configuration instead; But, make sure to update the IP address, gateway, and other network settings to match your local network.

```rust
pub fn init_net_stack(
    spawner: Spawner,
    net_device: cyw43::NetDriver<'static>,
    seed: u64,
) -> embassy_net::Stack<'static> {
    let config = Config::dhcpv4(Default::default());
    // Use static IP configuration instead of DHCP
    // let net_config = embassy_net::Config::ipv4_static(embassy_net::StaticConfigV4 {
    //     address: embassy_net::Ipv4Cidr::new(embassy_net::Ipv4Address::new(192, 168, 69, 2), 24),
    //     dns_servers: heapless::Vec::new(),
    //     gateway: Some(embassy_net::Ipv4Address::new(192, 168, 69, 1)),
    // });

    static RESOURCES: StaticCell<StackResources<5>> = StaticCell::new();
    let (stack, net_runner) = embassy_net::new(
        net_device,
        config,
        RESOURCES.init(StackResources::new()),
        seed,
    );

    spawner.spawn(unwrap!(net_task(net_runner)));

    stack
}
```

The `StackResources<5>` parameter specifies the memory reserved for socket slots. In this example, we allocate space for five socket slots. Since DHCP and DNS each require one socket slot, and our HTTP client also needs one, 3 is the minimum that would work. We reserve five socket slots in the project template to provide a little extra room for the networking examples later in the book without needing to change this value. You can experiment with different values. For example, after finishing this tutorial, try changing the value to 2 and see what happens.

The networking stack consists of two parts: a `Stack` and a `Runner`. The `Stack` is the handle that the rest of the application uses to access networking features such as TCP, UDP,and DHCP. Later in this tutorial, we will pass the `Stack` to the HTTP client so it can send and receive data over the network.

The `Runner` runs the networking stack in the background. It continuously processes incoming and outgoing packets, manages the network protocols.

The runner is started by spawning the following background task:

```rust
#[embassy_executor::task]
async fn net_task(mut runner: embassy_net::Runner<'static, cyw43::NetDriver<'static>>) -> ! {
    runner.run().await
}
```

## Joining a Wi-Fi Network

In this function, we connect the Pico W to your Wi-Fi network. It takes the Control handle returned by the CYW43 driver, along with your Wi-Fi network name (SSID) and password.

```rust
pub async fn join_wifi(control: &mut cyw43::Control<'static>, ssid: &str, password: &str) {
    while let Err(err) = control
        .join(ssid, cyw43::JoinOptions::new(password.as_bytes()))
        .await
    {
        info!("join failed: {:?}", err);
    }
}
```

## Bringing Up the Network

After successfully joining the Wi-Fi network, we need to wait until the networking stack is ready to use. This function first waits for the wireless link to come up, then waits until the network stack has a valid IP configuration.

In this example, we are using DHCP, so this means waiting for the router to assign an IP address to the Pico W. If you use a static IP configuration instead, the function waits until that configuration becomes active.

Once the network configuration is available, we print the assigned IP address, gateway, and DNS servers. 

```rust
/// Wait for link + DHCP.
pub async fn bring_up_stack(stack: embassy_net::Stack<'static>) {
    info!("waiting for link...");
    stack.wait_link_up().await;

    info!("waiting for DHCP...");
    stack.wait_config_up().await;

    if let Some(cfg) = stack.config_v4() {
        info!("IP: {:?}", cfg.address);
        info!("Gateway: {:?}", cfg.gateway);
        info!("DNS: {:?}", cfg.dns_servers);
    }

    // Custom DNS Servers
    // if let Some(mut cfg) = stack.config_v4() {
    //     cfg.dns_servers = heapless::Vec::from_slice(&[
    //         embassy_net::Ipv4Address::new(1, 1, 1, 1),
    //         embassy_net::Ipv4Address::new(8, 8, 8, 8),
    //     ])
    //     .expect("DNS server list exceeds heapless::Vec capacity");

    //     stack.set_config_v4(embassy_net::ConfigV4::Static(cfg));
    //     info!("Overrode DNS servers, keeping DHCP address/gateway");
    // }
}
```

The commented code is for overriding the DNS servers. I added it because, when I connected the Pico W to my home Wi-Fi router, I was unable to reach the `httpbin.org` website using the DNS servers assigned by DHCP. To work around the problem, I overrode the DNS servers with Cloudflare (`1.1.1.1`) and Google (`8.8.8.8`), which resolved the issue. You will most likely not need this code, but I have left it here in case you encounter a similar problem.
