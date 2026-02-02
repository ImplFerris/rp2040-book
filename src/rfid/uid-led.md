# Turn on LED When RFID UID Matches

In this section, we'll use the UID obtained in the previous chapter and hardcode it into our program. The LED will turn on only when the matching RFID tag is nearby; otherwise, it will remain off. When you bring the RFID tag close, the LED will light up. If you bring a different tag, like a key fob or any other RFID tag, the LED will turn off.

This should give you a basic authorized vs unauthorized feel. Simply checking the UID is not enough for real security, and we will go deeper into reading and authentication in the next chapter. For now, this is a simple and fun way to get started. You can also extend this example by adding an OLED display and showing Authorized or Unauthorized messages instead of using an LED.

## Getting Your Card's UID

Before writing the LED control logic, you need to know the UID of your RFID card.

Run the code from the previous chapter that prints the UID. When you bring your card near the reader, you will see output like this:
```sh
UID: 13 37 73 31
```

These are the four bytes you'll hardcode into the program as [0x13, 0x37, 0x73, 0x31].


## Logic

The logic used here is straightforward. The LED is kept off by default. The program continuously checks for an RFID tag. When a tag is detected, its UID is read and compared with the hardcoded UID. If they match, the LED is turned on. If they do not match, or if no tag is present, the LED remains off.

> [!Tip]
> If you are using a Pico W, the onboard LED is not connected directly to a GPIO pin. You may need to use the external LED circuit described earlier and connect the LED to GPIO 15, or follow the steps in the Quick Start section to access the onboard LED through the wireless chip.

```rust
// Replace the UID Bytes with your tag UID
const TAG_UID: [u8; 4] = [0x13, 0x37, 0x73, 0x31];

// On board LED only works for Pico not for Pico W
let mut led = Output::new(p.PIN_25, Level::Low);

// Using External LED, connected to GPIO 15
// let mut led = Output::new(p.PIN_15, Level::Low);

loop {
    led.set_low();

    if let Ok(atqa) = rfid.reqa() {
        if let Ok(uid) = rfid.select(&atqa) {
            if *uid.as_bytes() == TAG_UID {
                led.set_high();
                Timer::after_millis(500).await;
            }
        }
    }
    Timer::after_millis(100).await;
}
```



## Clone the existing project

You can clone (or refer) project I created and navigate to the `rfid-led` folder.

```sh
git clone https://github.com/ImplFerris/rp2040-projects
cd rp2040-projects/embassy/rfid/rfid-led/
```


## Light it Up

Flash the program onto the Pico as you normally would:

```sh
# if you are using debug probe
# cargo flash --release  
cargo embed --release

# if you are using BOOTSEL approach
cargo run --release
``` 

Once the program is running, bring the matching RFID tag near the reader and observe the onboard LED turning on. Move the tag away and the LED turns off. Try a different card or key fob and you will see that the LED does not turn on for non matching UIDs.
