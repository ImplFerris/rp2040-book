# Running the Program

Before flashing the program, make sure to provide your Wi-Fi credentials as environment variables. If you have already exported them in your shell, you can omit the SSID and PASSWORD prefix.

You can flash the program using:

```bash id="6twm5q"
SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD cargo embed --release
```

When I was running the program, I got a lot of log messages from the `cyw43` driver. They made it harder to find the assigned IP address in the debug output. To reduce this noise, I recommend setting the `cyw43` log level to `warn`:

```bash id="gms5yl"
SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD DEFMT_LOG="info,cyw43=warn" cargo embed --release
```

Once the program starts, the Pico W connects to your Wi-Fi network and starts the web server. The IP address assigned by your router will be printed to the debug console.

```sh
0.002473 [INFO ] Initializing the program
0.526870 [WARN ] BDC event, incomplete header
0.527011 [WARN ] BDC event, incomplete header
1.261423 [INFO ] IPv4: DOWN
1.261452 [INFO ] IPv6: DOWN
4.517621 [INFO ] waiting for link...
4.517747 [INFO ] link_up = true
4.517803 [INFO ] IPv4: DOWN
4.517822 [INFO ] IPv6: DOWN
4.518452 [INFO ] waiting for DHCP...
5.051267 [INFO ] IPv6: DOWN
5.051404 [WARN ] Number of DNS servers exceeds DNS_MAX_SERVER_COUNT, truncating list.
5.051630 [INFO ] IP: 192.168.0.103/24
5.051711 [INFO ] Gateway: Some(192.168.0.1)
5.051771 [INFO ] DNS: [8.8.8.8, 1.1.1.1]
5.051858 [INFO ] Stack is up!
5.054110 [INFO ] Running the server...
```

In my case, the Pico W's IP address was `192.168.0.103`. Your Pico W will most likely receive a different IP address.

Open your Pico W's IP address in any web browser connected to the same Wi-Fi network. You should see the web page, where you can turn the onboard LED on and off using the buttons.
