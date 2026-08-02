# Server Module

Open `server.rs` and add the following import statements:

```rust
use embassy_executor::Spawner;
use picoserve::extract::State;
use picoserve::response::File as PicoFile;
use picoserve::{
    AppBuilder, AppRouter, make_static,
    response::DebugValue,
    routing::{PathRouter, get, get_service, parse_path_segment},
};

// defmt Logging
use defmt::{info, unwrap};

use cyw43::Control;
use embassy_sync::{blocking_mutex::raw::CriticalSectionRawMutex, mutex::Mutex};
```

## Sharing the Wi-Fi Control

Our web server needs access to the `Control` object so it can turn the Pico W's onboard LED on and off. However, the web server runs multiple asynchronous tasks, so the `Control` object cannot be shared directly.

To safely share it between tasks, we wrap it inside an Embassy `Mutex`. We will create a type alias for a mutex that protects the `Control` object. The mutex ensures that only one task can access the `Control` object at a time.


```rust
type ControlMutex = Mutex<CriticalSectionRawMutex, Control<'static>>;
```

Next, we define a small wrapper around the mutex:

```rust
#[derive(Clone, Copy)]
struct SharedControl(&'static ControlMutex);
```

There is nothing special about this struct. It simply lets us use the name `SharedControl` instead of repeatedly writing `&'static ControlMutex`, making the code easier to read.


## App State

Next, we create a structure to hold the application's shared state:

```rust
#[derive(Clone, Copy)]

struct AppState {
    shared_control: SharedControl,
}
```

This holds the shared data that our request handlers can access. Right now, it contains only `SharedControl`, but you can add more shared data here as your application grows.

We will attach `AppState` to the router later using `.with_state(state)`, making it available to every request.
