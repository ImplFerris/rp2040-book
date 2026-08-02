# Building the Router

Now that we have created the application state, it is time to build our web application.

First, define the following structure:

```rust
struct AppProps {
    state: AppState,
}
```

`AppProps` acts as a container for the values used to build the router. In this example, it contains only the application state.

Next, implement the `AppBuilder` trait provided by picoserve:

```rust
impl AppBuilder for AppProps {
    type PathRouter = impl PathRouter;

    fn build_app(self) -> picoserve::Router<Self::PathRouter> {
        let Self { state } = self;

        picoserve::Router::new()
    }
}
```

The `build_app` function is responsible for constructing our web application. It returns a router that tells `picoserve` how to handle incoming HTTP requests.

At the moment, the router is empty. If we were to run the application now, every request would return a "Not Found" response because no routes have been registered yet.

## Serving the HTML Page

Let's begin by adding a route for the main HTML page:

```rust
impl AppBuilder for AppProps {
    type PathRouter = impl PathRouter;

    fn build_app(self) -> picoserve::Router<Self::PathRouter> {
        let Self { state } = self;

        picoserve::Router::new()
            .route("/", get_service(PicoFile::html(include_str!("index.html"))))
    }
}
```

The `.route` method registers a new route with the router.

The first argument, `"/"`, specifies the URL path. When a user enters the Pico W's IP address into a web browser, the browser requests this path.

The second argument tells `picoserve` how to handle that request. The `get_service` function creates a handler for HTTP GET requests, while `PicoFile::html` serves an HTML file.

## Serving CSS and JavaScript

The HTML page also references a stylesheet and a JavaScript file. We need to register routes for those files as well.

Add the following routes:

```rust
.route(
    "/index.css",
    get_service(PicoFile::css(include_str!("index.css"))),
)
.route(
    "/index.js",
    get_service(PicoFile::javascript(include_str!("index.js"))),
)
```

These routes work exactly like the HTML route. The only difference is the type of file being served. `PicoFile::css` serves a CSS file, while `PicoFile::javascript` serves a JavaScript file.

When the browser loads `index.html`, it automatically requests these files. The CSS file controls the appearance of the page, while the JavaScript file handles the button clicks.

## Adding the LED Control Route

The web page is now able to load successfully, but clicking the buttons will not do anything because the web server does not have a route to handle those requests.

When a button is clicked, the JavaScript code sends a request to the URL `/set-led/<boolean-value>`. To turn the LED on, it sends a request to `/set-led/true`. To turn it off, it sends a request to `/set-led/false`.

Add the following route:

```rust
.route(
    ("/set-led", parse_path_segment()),
    get(
        |led_is_on: bool, State(state): State<AppState>| async move {
            let SharedControl(control) = state.shared_control;

            info!("Setting led to {}", if led_is_on { "ON" } else { "OFF" });

            control.lock().await.gpio_set(0, led_is_on).await;

            DebugValue(led_is_on)
        },
    ),
)
```

The `parse_path_segment` function extracts the boolean value from the URL and passes it to the handler as the `led_is_on` parameter.

Unlike the previous routes, which simply served static files, this route executes a closure containing the logic to control the onboard LED. The closure receives both the boolean value from the URL and the application state.

Next, we extract the Wi-Fi control object from our `SharedControl` wrapper. We then lock the mutex to safely access the wireless chip, update the onboard LED using `gpio_set`, and return the new LED state as the HTTP response.
