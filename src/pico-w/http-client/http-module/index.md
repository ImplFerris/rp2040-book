# Creating the HTTP Client

Let's create a new module `http_client.rs` inside the `src` directory. Throughout the next few sections, we will gradually build the `fetch_json` function inside this file.

```text
src
├── main.rs
├── wifi.rs
└── http_client.rs
```

Let's first create a `FetchError` enum to represent the possible request errors.

```rust
#[derive(defmt::Format)]
pub enum FetchError {
    Request,
    Send,
    ReadBody,
}
```

Next, let's create the `fetch_json` function.

```rust
pub async fn fetch_json(
    stack: embassy_net::Stack<'static>,
) -> Result<(), FetchError> {
    //TODO
}
```

Before we can send an HTTP request, we need to create a few objects that work together. The `embassy-net` crate provides the TCP and DNS functionality, while the `reqwless` crate provides the HTTP client built on top of them.

In this example, we will use HTTP instead of HTTPS. I have also included the TLS code in case you want to experiment with HTTPS later.

Let's first import the required modules:

```rust
use embassy_net::dns::DnsSocket;
use embassy_net::tcp::client::{TcpClient, TcpClientState};

use reqwless::client::HttpClient;
// Uncomment these for TLS requests:
// use reqwless::client::{HttpClient, TlsConfig, TlsVerify};
```

HTTP runs on top of TCP. Before we can create the HTTP client, we first create the TCP and DNS clients it depends on. Inside the `fetch_json` function, create the following objects:

```rust

let client_state = TcpClientState::<1, 4096, 4096>::new();
let tcp_client = TcpClient::new(stack, &client_state);
let dns_client = DnsSocket::new(stack);

let mut http_client = HttpClient::new(&tcp_client, &dns_client);

// Uncomment these for TLS requests:
// let mut tls_read_buffer = [0; 16640];
// let mut tls_write_buffer = [0; 16640];

// Uncomment these for TLS requests:
// let tls_config = TlsConfig::new(seed, &mut tls_read_buffer, &mut tls_write_buffer, TlsVerify::None);
// let mut http_client = HttpClient::new_with_tls(&tcp_client, &dns_client, tls_config);
// let url = "https://httpbin.org/json";
```

The `TcpClientState` stores the state used by the TCP client. The generic parameters specify the maximum number of TCP connections that can be active at the same time, along with the sizes of the transmit and receive buffers for each connection. Since we only need one connection in this example, the first parameter is set to `1`.

Next, we create a `TcpClient` using the networking `Stack` that we initialized earlier. The TCP client is responsible for establishing TCP connections.  Then, we create a `DnsSocket`. It helps to resolve domain names into IP addresses.  Finally, we create the HTTP client that we can use to send HTTP requests.
