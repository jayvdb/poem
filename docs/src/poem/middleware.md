# Middleware

The middleware can do something before or after the request is processed.

`Poem` provides some commonly used middleware implementations.

- `AddData`

    Used to attach a status to the request, such as a token for authentication.

- `SetHeader`

    Used to add some specific HTTP headers to the response.

- `Cors`

    Used for Cross-Origin Resource Sharing.

- `Tracing`

  Use [`tracing`](https://crates.io/crates/tracing) to record all requests and responses.

## Custom middleware

It is easy to implement your own middleware, you only need to implement the `Middleware` trait, which is a converter to 
convert an input endpoint to another endpoint.

The following example creates a custom middleware that reads the value of the HTTP request header named `X-Token` and 
adds it as the status of the request.

```rust
use poem::{handler, web::Data, Endpoint, EndpointExt, Middleware, Request};

/// A middleware that extract token from HTTP headers.
struct TokenMiddleware;

impl<E: Endpoint> Middleware<E> for TokenMiddleware {
    type Output = TokenMiddlewareImpl<E>;
  
    fn transform(self, ep: E) -> Self::Output {
        TokenMiddlewareImpl { ep }
    }
}

/// The new endpoint type generated by the TokenMiddleware.
struct TokenMiddlewareImpl<E> {
    ep: E,
}

const TOKEN_HEADER: &str = "X-Token";

/// Token data
struct Token(String);

#[poem::async_trait]
impl<E: Endpoint> Endpoint for TokenMiddlewareImpl<E> {
    type Output = E::Output;
  
    async fn call(&self, mut req: Request) -> Self::Output {
        if let Some(value) = req
            .headers()
            .get(TOKEN_HEADER)
            .and_then(|value| value.to_str().ok())
        {
            // Insert token data to extensions of request.
            let token = value.to_string();
            req.extensions_mut().insert(Token(token));
        }
      
        // call the inner endpoint.
        self.ep.call(req).await
    }
}

#[handler]
async fn index(Data(token): Data<&Token>) -> String {
    token.0.clone()
}

// Use the `TokenMiddleware` middleware to convert the `index` endpoint.
let ep = index.with(TokenMiddleware);
```