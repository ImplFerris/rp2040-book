
# Using PIO for LED Blinking on Raspberry Pi Pico in Embedded Rust

So far, we wrote and understood our PIO assembly program in isolation. Now we need to turn that assembly into something that actually runs on the Raspberry Pi Pico using Embedded Rust and Embassy.

In Embassy, we do this using the `pio_asm!` macro. This macro lets us define our PIO assembly program directly in Rust. Once assembled, we load the instructions into the shared instruction memory of a PIO block and configure a state machine to execute them.

In this example, we will blink an external LED connected to GPIO 15 using PIO.


## Project from template

Generate a new project using the custom Embassy template.

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

## Binding the PIO Interrupt

PIO uses interrupts internally, so we must bind the PIO interrupt handler before using it. We bind interrupt `PIO0_IRQ_0` to the Embassy PIO interrupt handler.

```rust
bind_interrupts!(struct Irqs {
    PIO0_IRQ_0 => InterruptHandler<PIO0>;
});
```

At this point, we are not directly handling interrupts ourselves. Embassy uses this internally to manage PIO safely.


## Creating the PIO Block and State Machine

We initialize PIO block 0 and split it into its shared part and one state machine.

```rust
 let pio = p.PIO0;
let Pio {
    mut common,
    mut sm0,
    ..
} = Pio::new(pio, Irqs);
```

The common part is shared by all state machines in this PIO block. We use it to load programs into instruction memory and to configure GPIO pins for PIO use.

The sm0 value represents state machine 0. This is the state machine that will execute our PIO program.


## Preparing the Output GPIO Pin

Next, we convert GPIO 15 into a PIO-controlled pin.

```rust
let out_pin = common.make_pio_pin(p.PIN_15);
```
From this point onward, this pin is no longer controlled by the CPU GPIO peripheral. It will be driven directly by the PIO state machine.


## Defining the PIO Assembly Program

Now we define our PIO program using the `pio_asm!` macro. This is the same program we developed earlier, including instruction delays to produce a visible blink.

```rust
let prg = pio_asm!(
    "
    set pindirs, 1
    loop:
        set pins, 1 [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        set pins, 0 [30]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        nop [31]
        jmp loop
    "
);
```

This program runs forever. It drives the pin HIGH, waits, drives it LOW, waits, and then jumps back to the start of the loop. 

## Configuring the State Machine

Next, we configure the state machine that will run this program.

```rust
let mut cfg = Config::default();
cfg.use_program(&common.load_program(&prg.program), &[]);
cfg.set_set_pins(&[&out_pin]);
cfg.clock_divider = 65535u16.into();
sm0.set_config(&cfg);
```

Here is what we are doing step by step.

We load the assembled PIO program into the shared instruction memory using load_program. Then we tell the state machine which pin it should use for `set` instruction.

Finally, we set the clock divider to its maximum value. This slows down instruction execution so that the delays we added become visible on a real LED.

## Enabling the State Machine

Once the configuration is complete, we enable the state machine.

```rust
sm0.set_enable(true);
```

From this point onward, the PIO state machine starts executing the program independently of the main CPU.

## Main CPU Loop for Demonstration

To make it clear that the PIO is running independently, we keep the main CPU busy with a simple counter loop.

```rust
let mut counter: u8 = 0;
info!("Counter that running on main cpu - not related to the  PIO");
loop {
    info!("Count: {}", counter);

    counter = counter.wrapping_add(1);

    if counter == 0 {
        info!("Wrapped the counter");
    }

    Timer::after_millis(100).await;
}
```

This loop has nothing to do with the LED blinking. The LED continues blinking even if this loop is delayed or modified. This separation is one of the most important ideas behind PIO.


## Clone the existing project

You can clone (or refer) project I created and navigate to the `hello-blinky` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/pio/hello-blinky/
```

## Using a USB logic analyzer

By the way, I have been writing a series of blog posts while playing with a USB logic analyzer. I bought it recently and ended up using it to inspect almost everything.

I have created one blog post where I analyze this exact PIO blink program using a USB logic analyzer. In that post, I capture the real signal coming from the Pico and measure the HIGH and LOW timings on real hardware. I also calculate how much time the PIO instructions should take and compare that with what is captured using the logic analyzer.

You can check it out here:

[https://blog.implrust.com/posts/2026/02/logic-analyzer-raspberry-pi-pico-pio/](https://blog.implrust.com/posts/2026/02/logic-analyzer-raspberry-pi-pico-pio/)
