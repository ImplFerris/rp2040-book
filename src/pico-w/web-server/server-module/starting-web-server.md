# Starting the Web Server

So far, we have built the router that defines how incoming HTTP requests should be handled. The final step is to start the web server.

## Server Configuration

First, create a server configuration:

```rust
static CONFIG: picoserve::Config =  picoserve::Config::const_default().keep_connection_alive();
```

This creates a default `picoserve` configuration and enables HTTP keep-alive. By default, the server closes the TCP connection after sending each response. With keep-alive enabled, a client can reuse the same TCP connection for multiple HTTP requests instead of opening a new connection for each request.

Next, let's define the number of web server tasks:

```rust
const WEB_TASK_POOL_SIZE: usize = 4;
```

We will create a maximum of four web server tasks. This allows the Pico W to handle up to four HTTP connections at the same time. You can experiment with different values depending on your application's memory usage and performance requirements.

## The Web Server Task

Now, define the task that runs the web server:

```rust
#[embassy_executor::task(pool_size = WEB_TASK_POOL_SIZE)]
async fn web_task(
    task_id: usize,
    stack: embassy_net::Stack<'static>,
    app: &'static AppRouter<AppProps>,
) -> ! {
    let port = 80;
    let mut tcp_rx_buffer = [0; 1024];
    let mut tcp_tx_buffer = [0; 1024];
    let mut http_buffer = [0; 2048];

    picoserve::Server::new(app, &CONFIG, &mut http_buffer)
        .listen_and_serve(task_id, stack, port, &mut tcp_rx_buffer, &mut tcp_tx_buffer)
        .await
        .into_never()
}
```

Each web server task waits for incoming HTTP connections on port `80`. When a request arrives, `picoserve` uses the router we created earlier to determine which route matches the request and then executes the corresponding handler.

The receive, transmit, and HTTP buffers provide temporary working memory while processing client requests. The sizes used here are sufficient for this tutorial. If we serve larger web pages or handle larger HTTP requests, we may need to increase these buffer sizes.

Finally, we call `listen_and_serve` to start the web server. It waits for incoming TCP connections and processes one or more HTTP requests for each connection before waiting for the next one. Since our server is designed to run continuously, we use `into_never` to convert the return value into the `!` (never) type.

## Starting the Server

Finally, add the following function:

```rust
pub fn start(
    spawner: Spawner,
    stack: embassy_net::Stack<'static>,
    control: Control<'static>,
) {
    let shared_control = SharedControl(make_static!(ControlMutex, Mutex::new(control)));

    let app = make_static!(
        AppRouter<AppProps>,
        AppProps {
            state: AppState { shared_control }
        }
        .build_app()
    );

    info!("Running the server...");

    for task_id in 0..WEB_TASK_POOL_SIZE {
        spawner.spawn(unwrap!(web_task(task_id, stack, app)));
    }
}
```

In this function, we first prepare everything needed by the web server. We create the shared `Control` object, build the router, and store both in static memory so they can be shared by all the web server tasks.

Once everything is ready, we spawn four web server tasks. Each task waits for incoming HTTP connections and uses the router we built earlier to process requests.

At this point, our web server is ready to run. The only thing left is to call `start` from `main.rs`:

```rust
server::start(spawner, stack, control);
```
