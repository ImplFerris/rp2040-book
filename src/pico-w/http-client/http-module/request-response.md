# Request and Response

In this section, we will use the HTTP client to send a request and then convert the received response bytes into a string reference.

## Sending an HTTP GET Request

We need to specify what HTTP method we are going to use in the request function, so we will import the `Method` enum:

```rust
use reqwless::request::Method;
```

Next, inside the `fetch_json` function, add the following code:

```rust
info!("connecting to {}", &url);

let mut http_request = match http_client.request(Method::GET, url).await {
    Ok(req) => req,
    Err(e) => {
        error!("Failed to make HTTP request: {:?}", e);
        return Err(FetchError::Request);
    }
};
```

With the help of the `request` method, we prepare an HTTP request. The function takes two arguments: the HTTP method and the request URL. In this example, we create a GET request to `httpbin.org`.  Calling `request` does not send the request immediately. Instead, it returns a request object that we will send in the next step.

Next, create a receive buffer to store the server's response, and then send the request:

```rust
let mut rx_buffer = [0; 4096];

let response = match http_request.send(&mut rx_buffer).await {
    Ok(resp) => resp,
    Err(e) => {
        error!("Failed to send HTTP request: {:?}", e);
        return Err(FetchError::Send);
    }
};
```

Finally, we call the `send` method to transmit the request to the server. If the request completes successfully, it returns a `response` object that we can use to access the HTTP response returned by the server.

## Reading the HTTP Response

Now, let's continue the `fetch_json` function and add the following code:

```rust
info!("Response status: {}", response.status.0);

let body_bytes = match response.body().read_to_end().await {
    Ok(b) => b,
    Err(_e) => {
        error!("Failed to read response body");
        return Err(FetchError::ReadBody);
    }
};
```

First, we print the HTTP status code. If the request completed successfully, the status code should be `200`.  Next, we call the `body` method to access the response body, and then call the `read_to_end` method to read the entire response into the `body_bytes` variable.
