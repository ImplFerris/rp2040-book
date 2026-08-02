# Setting Up the Project

The project template we have been using so far is intended for general Raspberry Pi Pico development. Since networking on the Pico W requires additional setup, we will use a different project template for this tutorial.

The template already includes support for the CYW43439 wireless chip, the required firmware files, and the project configuration needed for networking.

Create a new project with:

```bash
cargo generate --git https://github.com/ImplFerris/pico-w-template.git --tag v0.1.1
```

Alternatively, you can clone the template and generate the project from your local copy:

```bash
git clone https://github.com/ImplFerris/pico-w-template.git

# Replace /path/to/ with the directory where you cloned the repository.
cargo generate /path/to/pico-w-template --tag v0.1.1
```

When prompted, enter a project name as usual and enable the `defmt` option. Throughout this tutorial, we will use `defmt` for logging, so you will need the Raspberry Pi Debug Probe.  If you do not have one, you can disable the `defmt` option and use USB serial logging approach instead.

## Project Structure

In the template, I have moved the Wi-Fi initialization code into a separate `wifi` module because it is boilerplate code. This keeps `main.rs` small and allows us to focus on the application logic.

```
.
├── build.rs
├── Cargo.toml
├── Embed.toml
├── firmware
│   ├── 43439A0.bin
│   ├── 43439A0_clm.bin
│   └── nvram_rp2040.bin
├── memory.x
├── src
│   ├── main.rs
│   └── wifi.rs
```

You will also notice an additional firmware file named `nvram_rp2040.bin`. Recent versions of the `cyw43` crate require this file when initializing the CYW43439 wireless chip.

<!-- > TODO: Explain the purpose of `nvram_rp2040.bin`. -->

I recommend taking a few minutes to browse through `main.rs` and `wifi.rs` to get familiar with the project structure. In the following sections, we will go through the code in detail.
 
## Wi-Fi Credentials

I configured the template to read the Wi-Fi network name (SSID) and password from environment variables at compile time.

```rust
const WIFI_NETWORK: &str = env!("SSID");
const WIFI_PASSWORD: &str = env!("PASSWORD");
```

Whenever you build or flash the program, make sure to provide these environment variables. Alternatively, you can export them in your shell before running Cargo.

```sh
SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD cargo build --release
# for flashing
# SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD cargo embed --release

# Reduce cyw43 log noise
SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD DEFMT_LOG="info,cyw43=warn" cargo build --release
```

## The Main

Before we send our first HTTP request, let's first look at the current `main.rs`.

At this point, the program contains only the boilerplate required to initialize the CYW43 driver, create the networking stack, connect to a Wi-Fi network, and wait until the network is ready. We have not added any HTTP client code yet.

```rust

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    info!("Initializing the program");

    let (net_device, mut control) = wifi::init_cyw43(
        spawner, p.PIO0, p.DMA_CH0, p.PIN_23, p.PIN_24, p.PIN_25, p.PIN_29,
    )
    .await;

    let mut rng = RoscRng;
    let seed = rng.next_u64();

    let stack = wifi::init_net_stack(spawner, net_device, seed);

    wifi::join_wifi(&mut control, WIFI_NETWORK, WIFI_PASSWORD).await;
    wifi::bring_up_stack(stack).await;
    info!("Stack is up!");

    loop {
        Timer::after_secs(1).await;
    }
}
```

In the next chapter, we will look at how the wifi module works.
