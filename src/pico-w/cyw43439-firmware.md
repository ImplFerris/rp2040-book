{{#title CYW43439 Firmware}}

# CYW43439 Firmware

With the communication interface in place, the next step is to load the firmware onto the CYW43439. Until this firmware starts running, the wireless chip cannot process requests from the RP2040, such as controlling the onboard LED or performing Wi-Fi operations.

The firmware supplied with the Pico W is provided as a prebuilt binary by Infineon and is not open source.  These firmware files are also available from the Embassy project:

[https://github.com/embassy-rs/embassy/tree/main/cyw43-firmware](https://github.com/embassy-rs/embassy/tree/main/cyw43-firmware)

## Firmware Files

We use two firmware files throughout this book.

- **43439A0.bin** is the main executable firmware that runs on the CYW43439.
- **43439A0_clm.bin** contains the Country Locale Matrix (CLM). Rather than executable code, it contains regulatory information that tells the wireless firmware which Wi-Fi channels and transmit settings are permitted in different countries.

Both files are required before the CYW43439 can be fully initialized.

## Loading the Firmware

There are two common ways to provide these firmware files to your application.

### Option 1: Flash the Firmware Separately

The firmware can be programmed into fixed locations in the RP2040's external flash memory. During development, this is often the preferred approach because the firmware only needs to be flashed once. Subsequent application builds become much smaller and flash significantly faster.

Flash the firmware using probe-rs:

```bash
probe-rs download 43439A0.bin \
  --binary-format bin \
  --chip RP2040 \
  --base-address 0x10100000

probe-rs download 43439A0_clm.bin \
  --binary-format bin \
  --chip RP2040 \
  --base-address 0x10140000
```

The firmware can then be accessed from Rust using slices that point to those flash addresses.

```rust
let fw = unsafe {
    core::slice::from_raw_parts(0x1010_0000 as *const u8, 230_321)
};

let clm = unsafe {
    core::slice::from_raw_parts(0x1014_0000 as *const u8, 4_752)
};
```

### Option 2: Embed the Firmware

The second approach is to embed the firmware directly into the application binary using `include_bytes!`.

```rust
let fw: &[u8] = include_bytes!("../firmware/43439A0.bin");
let clm: &[u8] = include_bytes!("../firmware/43439A0_clm.bin");
```

This produces a self-contained application because the firmware is packaged together with your program. The downside is that every build includes the firmware, increasing both the application size and flashing time.
