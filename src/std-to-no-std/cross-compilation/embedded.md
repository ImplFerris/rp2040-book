# Compiling for Microcontroller

Now let's talk about embedded systems. When it comes to compiling Rust code for a microcontroller, things work a little differently from normal desktop systems. Microcontrollers don’t usually run a full operating system like Linux or Windows. Instead, they run in a minimal environment, often with no OS at all. This is called a bare-metal environment.

Rust supports this kind of setup through its **no_std** mode. In normal Rust programs, the standard library (`std`) handles things like file systems, threads, heap allocation, and I/O. But none of those exist on a bare-metal microcontroller. So instead of std, we use a much smaller `core` library, which provides only the essential building blocks.

## The Target Triple for Pico

The Raspberry Pi Pico is based on the RP2040 microcontroller. The RP2040 uses dual-core Arm Cortex-M0+ processors.

For RP2040, we use the following target:

```text
thumbv6m-none-eabi
```

Let's break this down:

- **Architecture (thumbv6m)**: Cortex-M0+ uses the ARMv6-M Thumb instruction set.
- **Vendor (none)**: No specific vendor designation.
- **OS (none)**: No operating system - it's bare-metal.
- **ABI (eabi)**: Embedded Application Binary Interface, the standard calling convention for embedded ARM systems.

To install and use this target:
```bash
rustup target add thumbv6m-none-eabi
cargo build --target thumbv6m-none-eabi
```

## Cargo Config

In the quick start sections, you may have noticed that we never manually passed the `--target` flag when running Cargo commands. This works because the target is already configured in the project.

Cargo allows you to store build-related settings in a .cargo/config.toml file. One of these settings is the default compilation target.

To configure this for RP2040, create a .cargo directory in the project root and add a config.toml file with the following content:

```toml
[build]
target = "thumbv6m-none-eabi"
```

Now you don't have to pass `--target` every time. Cargo will use this automatically.
