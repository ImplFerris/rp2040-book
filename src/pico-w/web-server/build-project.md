# Seting Up the Project

Let's create a new project with Pico W Template:

```bash
cargo generate --git https://github.com/ImplFerris/pico-w-template.git --tag v0.1.1
```

This template includes the boilerplate code required to connect the Pico W to a Wi-Fi network. We will build our web server on top of it.

We will now start modifying our project. The implementation will be based on the [`set_pico_w_led` example](https://github.com/sammhicks/picoserve/tree/612a2f2b730f97d6f773815313c7a566a2c1ea3d/examples/embassy/set_pico_w_led) from the picoserve project.

## Additional Crates

Add the following crates to your `Cargo.toml` file:

```toml
picoserve = { version = "0.19.0", features = ["embassy"] }
embassy-sync = "0.8.0"
```

As we mentioned earlier, `picoserve` is the main crate that will handle incoming web requests. The `embassy-sync` provides synchronization primitives that work with Embassy's asynchronous runtime.

## Nightly Rust

So far, we have been using the stable Rust toolchain. For this chapter, however, we need to switch to a nightly toolchain because the current version of `picoserve` depends on an experimental Rust language feature that is not yet available on stable Rust.

Create a `rust-toolchain.toml` file in the project root and add the following lines:

```toml
[toolchain]
targets = ["thumbv6m-none-eabi"]
channel = "nightly-2026-04-14"
```

Next, add the following line to the top of your `main.rs` file to enable the required feature:

```rust
#![feature(impl_trait_in_assoc_type)]
```

## Server Module

Create a new file named `server.rs` inside the `src` folder. We will place all the web server code in this file to keep `main.rs` clean.

Next, add the following line to `main.rs`:

```rust
mod server;
```

## Web Files

Our web server serves three static files: `index.html`, `index.css`, and `index.js`. Download these files from the following links and place them inside the `src` folder, alongside `main.rs` and `server.rs`.

[https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.html](https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.html)

[https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.css](https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.css)

[https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.js](https://raw.githubusercontent.com/ImplFerris/pico-w-projects/refs/heads/main/embassy/web-server/src/index.js)

Your project structure should now look like this:

```text
src
├── index.css
├── index.html
├── index.js
├── main.rs
├── server.rs
└── wifi.rs
```

The HTML file defines a simple web page with two buttons to control the onboard LED. When a button is clicked, the JavaScript sends an HTTP request to our Pico W to turn the LED on or off. The CSS file simply styles the web page to make it look better.
