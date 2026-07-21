# Parsing the JSON Response

In this section, we will parse the JSON response returned by the server.  To parse JSON, we will use the `serde` and `serde-json-core` crates. Add the following dependencies to your `Cargo.toml` file:

```toml
serde = { version = "1.0.203", default-features = false, features = ["derive"] }
serde-json-core = "0.5.1"
```

Next, import the required modules:

```rust
use serde::Deserialize;
use serde_json_core::from_slice;
use defmt::Debug2Format;
```

Before we write further code, let's first look at the JSON returned by `http://httpbin.org/json`. Open the URL in your browser, and you will see a response similar to the following:

```json
{
  "slideshow": {
    "author": "Yours Truly", 
    "date": "date of publication", 
    "slides": [
      {
        "title": "Wake up to WonderWidgets!", 
        "type": "all"
      }, 
      {
        "items": [
          "Why <em>WonderWidgets</em> are great", 
          "Who <em>buys</em> WonderWidgets"
        ], 
        "title": "Overview", 
        "type": "all"
      }
    ], 
    "title": "Sample Slide Show"
  }
}
```

We are going to just print the `author` and `title` fields from this JSON response. Since we only need these two fields, we do not have to define every field in our Rust structs.

The `Deserialize` derive allows `serde` to convert JSON data into Rust structures automatically.

Notice that the JSON object at the root level contains a single field named `slideshow`. To match this structure, create a top-level `HttpBinResponse` struct with a `slideshow` field:

```rust
#[derive(Deserialize)]
struct HttpBinResponse<'a> {
    #[serde(borrow)]
    slideshow: SlideShow<'a>,
}
```
The `slideshow` field maps to another struct named `SlideShow`, which represents the `slideshow` object in the JSON.  

```rust
#[derive(Deserialize)]
struct SlideShow<'a> {
    author: &'a str,
    title: &'a str,
}
```

The `SlideShow` object contains several fields, including author, date, slides, and title. We only define the `author` and `title` fields because those are the only values we are interested. The remaining fields are ignored automatically during deserialization.

Now, add the following code to parse the JSON response:

```rust
match from_slice::<HttpBinResponse>(body_bytes) {
    Ok((output, _used)) => {
        info!("Successfully parsed JSON response!");
        info!("Slideshow title: {:?}", output.slideshow.title);
        info!("Slideshow author: {:?}", output.slideshow.author);
    }
    Err(e) => {
        error!("Failed to parse JSON response: {}", Debug2Format(&e));

        // Log preview of response for debugging
        let preview = if body_bytes.len() > 200 {
            &body_bytes[..200]
        } else {
            body_bytes
        };
        match core::str::from_utf8(preview) {
            Ok(text) => info!("Response preview: {:?}", text),
            Err(_) => info!("Response preview contains non-UTF-8 data"),
        }
    }
}
```

Next, we call the `from_slice` function to deserialize the JSON into our `HttpBinResponse` struct.

If the JSON is parsed successfully, we log a success message and print the slideshow title and author.

If parsing fails, we log the error and print a preview of the response body. While writing this chapter, I opened the URL in my browser and found that "httpbin.org" site was temporarily returning an HTML error page instead of JSON. In such cases, the response preview can help you quickly identify the problem.


## Running the Program

Now, call the `fetch_json` function from `main`:

```rust
if let Err(e) = http_client::fetch_json(stack).await {
    info!("Error fetching {}", e);
}
```

Build and run the program. If everything is configured correctly, the Pico W connects to the Wi-Fi network, sends an HTTP GET request to `httpbin.org`, and prints the slideshow title and author returned in the JSON response.

Don't forget to provide your Wi-Fi credentials as environment variables when flashing the program, or make sure they have already been exported in your shell:

```sh
SSID=YOUR_WIFI PASSWORD=YOUR_WIFI_PASSWORD cargo embed --release
```
