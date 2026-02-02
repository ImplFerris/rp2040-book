# Project Template with `cargo-generate`

"cargo-generate is a developer tool to help you get up and running quickly with a new Rust project by leveraging a pre-existing git repository as a template."

Read more about [here](https://github.com/cargo-generate/cargo-generate).
 
## Prerequisites

Before starting, ensure you have the following tools installed:

- [Rust](https://www.rust-lang.org/tools/install)
- [cargo-generate](https://github.com/cargo-generate/cargo-generate) for generating the project template.


Install the OpenSSL development package first because it is required by cargo-generate:
```sh
sudo apt install  libssl-dev
```

You can install `cargo-generate` using the following command:

```sh
cargo install cargo-generate
```

## Generate the Project
Run the following command to generate the project from the template:

```sh
cargo generate --git https://github.com/ImplFerris/rp2040-embassy-template.git --tag v0.1.4
```

This will prompt you to answer a few questions:
Project name: Name your project.
Enable defmt logging (requires a debug probe)?: true or false

The code structure may look like this:

`src/main.rs`: Contains the default blink logic.

`Cargo.toml`: Includes dependencies for the selected HAL.

