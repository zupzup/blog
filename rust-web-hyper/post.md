*This post was originally posted on the [LogRocket](https://blog.logrocket.com/a-minimal-web-service-in-rust-using-hyper/) blog on 20.10.2020 and was cross-posted here by the author.*

In my experience, when building web services, simpler is better. It’s tempting to take a fully featured, heavyweight web framework, drop it in, and use its convention-over-configuration approach to just “get stuff done”. Many developers have used this approach successfully, myself included.

However, I noticed after a while that the hidden complexity — which is huge — has some drawbacks. These drawbacks can range from degraded performance, to time lost debugging issues in the huge network of transitive dependencies, to simply not knowing what’s going on. Likewise, writing everything from scratch is not optimal. It’s extremely time-consuming there is potential to introduce errors at every step.

I’m not saying that widely used open-source web frameworks are error-free — far from it. But there are, at the least, more eyes on them and more stakeholders actively finding and fixing bugs.

In my opinion, there is a happy medium. The goal is to keep complexity low, while retaining most of the convenience and development speed you get from having everything in one dependency.

This sweet spot might be different for different people because, depending on your experience, what you’re comfortable writing yourself or using a microdependency for will vary. But the general approach — to take multiple, small(er) libraries and build a minimal system out of them — has worked well for me in the past. One advantage is that the dependencies are small enough that you can actually, in finite time, go to their repository to read, understand, and, if necessary, fix them. What’s more, you’ll have a better overview of what’s in the codebase (i.e., what can go wrong) and what isn’t, which is difficult with huge frameworks. This enables you to actually tailor the system to the problem you’re solving.

You can define base-level APIs exactly as you need them to build your system without having to fight the framework to get things done. This requires a certain level of experience to get right, but if you’re willing to spend time wrestling with frameworks, it’s certainly worth the effort.

In this tutorial, we’ll show you how to build a Rust web service without using a web framework.

We won’t be building everything from scratch, though. For our HTTP server, we’ll use [hyper](https://hyper.rs/), which uses the tokio runtime underneath. Neither of these libraries is the most lightweight or minimal of options, but both are widely used and the concepts described here will apply regardless of the libraries used.

We’ll build a basic web server with a somewhat flexible routing API and a few sample handlers to show it off. By no means will our finished product be ready for production to use as-is, but by the end of the tutorial, you should have a clear idea of how you can extend it to get there.

## Setup

To follow along, you’ll need a recent Rust installation (1.39+) and a tool to send HTTP requests, such as cURL.

First, create a new Rust project.

```bash
cargo new rust-minimal-web-example
cd rust-minimal-web-example
```

Next, edit the `Cargo.toml` file and add the following dependencies.

```toml
[dependencies]
futures = { version = "0.3.6", default-features = false, features = ["async-await"] }
hyper = "0.13"
tokio = { version = "0.2", features = ["macros", "rt-threaded"] }
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0"
route-recognizer = "0.2"
bytes = "0.5"
async-trait = "0.1"
```

That’s quite a few dependencies for a “minimal” web application. What’s up with that?

Since we’re not using a full web framework but trying to build our own system composed of several micro-libraries, the overall complexity is still lower, even if the number of direct dependencies goes up.

It’s not necessarily the number of direct dependencies we’re worried about, but the number of transitive dependencies and the amount of code that glues them together.

Since we’re using `hyper` as an async HTTP server, we also need an async runtime. In this example, we use tokio, but we could also use a lighter-weight solution, such as [smol](https://github.com/stjepang/smol).

The `serde` and `serde_json` dependencies are necessary for handling incoming JSON. We could probably get away with using [nanoserde](https://github.com/not-fl3/nanoserde) as well if we wanted to minimalize everything. The `route-recognizer` crate is a very small, lightweight router that can handle paths with parameters such as `/product/:product_id` as well.

For the remaining libraries - namely, `bytes`, `async-trait`, and `futures` - we need them to build our router. They’re very likely transitive dependencies of whatever we’re using already anyway, so they don’t add any weight and they’re not particularly heavy. In the case of `futures`, we could also use the [futures-lite](https://github.com/stjepang/futures-lite) crate.

## Handler API

Let’s build this from the bottom-up and look at the API we want for our handlers first. Then, we’ll implement a router and, at the end, put everything together.

We’ll create three handlers in this example:

* `GET /test`, a basic handler that returns a string to show it works
* `POST /send`, a handler expecting a JSON payload, which returns an error if it isn’t valid
* `GET /params/:some_param`, a simple handler to show off how we handle path parameters

Let’s implement those in `handler.rs` to see the API we’d like to create for them.

```rust
use crate::{Context, Response};
use hyper::StatusCode;
use serde::Deserialize;

pub async fn test_handler(ctx: Context) -> String {
    format!("test called, state_thing was: {}", ctx.state.state_thing)
}

#[derive(Deserialize)]
struct SendRequest {
    name: String,
    active: bool,
}

pub async fn send_handler(mut ctx: Context) -> Response {
    let body: SendRequest = match ctx.body_json().await {
        Ok(v) => v,
        Err(e) => {
            return hyper::Response::builder()
                .status(StatusCode::BAD_REQUEST)
                .body(format!("could not parse JSON: {}", e).into())
                .unwrap()
        }
    };

    Response::new(
        format!(
            "send called with name: {} and active: {}",
            body.name, body.active
        )
        .into(),
    )
}

pub async fn param_handler(ctx: Context) -> String {
    let param = match ctx.params.find("some_param") {
        Some(v) => v,
        None => "empty",
    };
    format!("param called, param was: {}", param)
}
```

The `test_handler` is very simple, but we’ve already seen one important concept: the `Context`. To use request state (such as the body, query parameters, headers etc.) within a handler, we need some way to get it in there.

Also, depending on how we want to architect the system, we might want to make certain shared systems (an HTTP client or a database repository) available to handlers. In this example, we’ll use a `Context` object to encapsulate these things. In this case, we return the content of the `ctx.state.state_thing` variable, which is a dummy version of some actual application state. We’ll look at how it’s implemented later on, but it holds every bit of information the handler might need to do its work.

For the `send_handler`, we already see `Context` in action. We expect a `SendRequest` JSON payload in this handler, so we use the `ctx.body_json()` method to parse the request body to a `SendRequest`. If this goes wrong, we simply return a 400 error.

An interesting difference between `test_handler` and `send_handler` is the return type. We want to be able to return values of different types. In the most basic case, a `String`, or a `&'static str` will do. In other cases — as in our case — we’d want to return a `Result<>` or a raw `Response`.

The third handler, `param_handler`, shows off how we can use `Context` to get to path parameters, which are defined in the route. All handlers are `async` functions, but we could also create a handler using `impl Future` by hand.

In `main.rs`, we can add the definition for the `Context` and some helper types.

```rust
use route_recognizer::Params;

type Response = hyper::Response<hyper::Body>;
type Error = Box<dyn std::error::Error + Send + Sync + 'static>;

#[derive(Clone, Debug)]
pub struct AppState {
    pub state_thing: String,
}

#[derive(Debug)]
pub struct Context {
    pub state: AppState,
    pub req: Request<Body>,
    pub params: Params,
    body_bytes: Option<Bytes>,
}

impl Context {
    pub fn new(state: AppState, req: Request<Body>, params: Params) -> Context {
        Context {
            state,
            req,
            params,
            body_bytes: None,
        }
    }

    pub async fn body_json<T: serde::de::DeserializeOwned>(&mut self) -> Result<T, Error> {
        let body_bytes = match self.body_bytes {
            Some(ref v) => v,
            _ => {
                let body = to_bytes(self.req.body_mut()).await?;
                self.body_bytes = Some(body);
                self.body_bytes.as_ref().expect("body_bytes was set above")
            }
        };
        Ok(serde_json::from_slice(&body_bytes)?)
    }
}
```

Here, we define `Response` to avoid some typing and `Error`, which is a generic error type. In practice, we’d probably want to use a custom error type to propagate errors throughout our application.

Next, we define our `AppState` struct. In this example, this simply holds the aforementioned dummy application state. This `AppState` could, for example, also hold shared references to a cache, a database repository, or other application-wide things you’d want your handlers to access.

The `Context` itself is just a struct containing the `AppState`, the incoming `Request`, the path parameters if there are any, and `body_bytes`. It exposes one function called `body_json`, which sets `body_bytes` and tries to parse it to the given type.

We memoize `body_bytes` here because it’s possible we’d want to access the body multiple times during the request’s life cycle. This way, we only have to read and store it once (for example, in a middleware).

## Routing

Now that we know what our handler API should look like, we need to build a routing mechanism to accommodate it.

The approach shown here is a simplified (and likely worse) version of what the [tide](https://github.com/http-rs/tide) framework uses for routing. It would be nice to have a simple API to define routes:

```rust
let mut router: Router = Router::new();
router.get("/test", Box::new(handler::test_handler));
router.post("/send", Box::new(handler::send_handler));
router.get("/params/:some_param", Box::new(handler::param_handler));
```

We create a `router` and use helper methods for the different HTTP methods to add handlers to it. In this example, we wrap the handler functions in a `Box`, which is a pointer type for heap allocations.

We could avoid this by creating another trait, which hides this bit of complexity from the caller, but this would have made the whole example more complex, so I left it out.

Next, let’s look at the router implementation in `router.rs`. We’ll start with the definition of the router and some dependencies.

```rust
use crate::{Context, Response};
use async_trait::async_trait;
use futures::future::Future;
use hyper::{Method, StatusCode};
use route_recognizer::{Match, Params, Router as InternalRouter};
use std::collections::HashMap;

pub struct Router {
    method_map: HashMap<Method, InternalRouter<Box<dyn Handler>>>,
}
```

The router is simply a struct, which holds a `method_map` internally. This is just a `HashMap`, which separates the registered routes by their HTTP method. In practice, we would probably use a more efficient `HashMap` implementation for this, such as fnv, but this is fine for our example.

The map’s entries are of type `InternalRouter`, which is the router created using `route_recognizer`, and each of those routers holds values of type `Box<dyn Handler>`, which we’ll look at next.

```rust
#[async_trait]
pub trait Handler: Send + Sync + 'static {
    async fn invoke(&self, context: Context) -> Response;
}

#[async_trait]
impl<F: Send + Sync + 'static, Fut> Handler for F
where
    F: Fn(Context) -> Fut,
    Fut: Future + Send + 'static,
    Fut::Output: IntoResponse,
{
    async fn invoke(&self, context: Context) -> Response {
        (self)(context).await.into_response()
    }
}
```

This is a bit more complex. To put references to our handler functions inside the router, and so we can pass the request `Context` to it, we define the `Handler` trait.

Then, we implement this `Handler` trait for `F`, where `F` is a function taking a `Context` and returning a `Future` (like our handlers are supposed to).

Also, we define that these futures have a type implementing `IntoResponse` as return type.

The `IntoResponse` trait looks like this:

```rust
pub trait IntoResponse: Send + Sized {
    fn into_response(self) -> Response;
}

impl IntoResponse for Response {
    fn into_response(self) -> Response {
        self
    }
}

impl IntoResponse for &'static str {
    fn into_response(self) -> Response {
        Response::new(self.into())
    }
}

impl IntoResponse for String {
    fn into_response(self) -> Response {
        Response::new(self.into())
    }
}
```

This simple trait is the reason we’re able to return `String` and `'static str` and `Response` from our handlers. We could add arbitrary other types here, such as `Result<T,E>`, which would automatically handle and return the error in a nice way.

But back to the `Handler` trait. The trait only includes one function, `invoke`, which is an indirection to pass a `Context` to our handlers. The `invoke` function simply calls self (the handler function) with the given context, awaits the future, and turns the result into a response.

With this in place, we can add any async function, which takes a `Context` and returns an `IntoResponse` to our router. Nice.

Let’s look at the implementation of `Router` next.

```rust
pub struct RouterMatch<'a> {
    pub handler: &'a dyn Handler,
    pub params: Params,
}


impl Router {
    pub fn new() -> Router {
        Router {
            method_map: HashMap::default(),
        }
    }

    pub fn get(&mut self, path: &str, handler: Box<dyn Handler>) {
        self.method_map
            .entry(Method::GET)
            .or_insert_with(InternalRouter::new)
            .add(path, handler)
    }

    pub fn post(&mut self, path: &str, handler: Box<dyn Handler>) {
        self.method_map
            .entry(Method::POST)
            .or_insert_with(InternalRouter::new)
            .add(path, handler)
    }

    pub fn route(&self, path: &str, method: &Method) -> RouterMatch<'_> {
        if let Some(Match { handler, params }) = self
            .method_map
            .get(method)
            .and_then(|r| r.recognize(path).ok())
        {
            RouterMatch {
                handler: &**handler,
                params,
            }
        } else {
            RouterMatch {
                handler: &not_found_handler,
                params: Params::new(),
            }
        }
    }
}

async fn not_found_handler(_cx: Context) -> Response {
    hyper::Response::builder()
        .status(StatusCode::NOT_FOUND)
        .body("NOT FOUND".into())
        .unwrap()
}
```

The `new` function creates the `Router` with an empty `method_map`. Then, we add the helpers for `get` and `post`. For brevity, the helpers for `put`, `delete`, etc. are omitted here, but they’re just more of the same.

In these helpers, we see if the `method_map` already has an entry for the given `Method` and, if not, create a new router inside it. In any case, an entry with the `path` and the `handler` function is added to the `route_recognizer` router inside.

This is how we add routes, but how do we do the actual routing? This is where the `route` function comes in. It’s called with the incoming `path` and the request `Method` and returns a `RouterMatch`. The `RouterMatch` struct holds a reference to the returned handler function and the path parameters calculated by `route_recognizer`, if there are any.

Basically, the `method_map` is asked for an entry with the given HTTP method and the underlying internal router is asked for a route with the given path. The internal router returns a `Match`. The `Match` includes the handler and path parameters, which are returned and wrapped in a `RouterMatch`. The `&**handler` syntax might look a bit strange, but we need it here; because we get a `&Box<dyn Handler>` and want to transform it to a `&'a dyn Handler`, we need to dereference the `&` and the `Box` and reference the outcome again using `&`. If nothing is found, we simply return a `404 NOT FOUND`.

## Putting it all together

The router is in place, the `Context` is there, and our handlers are ready to go. The only thing left to do  is wire everything together and hope it works.

First, the `main` function in `main.rs`:

```rust
use bytes::Bytes;
use hyper::{
    body::to_bytes,
    service::{make_service_fn, service_fn},
    Body, Request, Server,
};
use route_recognizer::Params;
use router::Router;
use std::sync::Arc;

mod handler;
mod router;

#[tokio::main]
async fn main() {
    let some_state = "state".to_string();

    let mut router: Router = Router::new();
    router.get("/test", Box::new(handler::test_handler));
    router.post("/send", Box::new(handler::send_handler));
    router.get("/params/:some_param", Box::new(handler::param_handler));

    let shared_router = Arc::new(router);
    let new_service = make_service_fn(move |_| {
        let app_state = AppState {
            state_thing: some_state.clone(),
        };

        let router_capture = shared_router.clone();
        async {
            Ok::<_, Error>(service_fn(move |req| {
                route(router_capture.clone(), req, app_state.clone())
            }))
        }
    });

    let addr = "0.0.0.0:8080".parse().expect("address creation works");
    let server = Server::bind(&addr).serve(new_service);
    println!("Listening on http://{}", addr);
    let _ = server.await;
}
```

We create a `Router` and add some routes. Then, we put the `router` inside of an `Arc` - a smart pointer that can be shared across threads — and use the `make_service_fn` function provided by `hyper` to define what should happen for incoming requests.

Inside the service closure, we create the `AppState`, cloning the predefined dummy string inside. We also need to clone our shared router inside this closure before passing it into the `async` block. This is necessary because the insides of the `async` block can and will be executed at a later time and on many different threads, so we need to make sure everything we give it lives long enough.

Inside the `async` block, we use the `service_fn` to define what should happen for every incoming request. We provide a closure, move our router and state inside it, and call the `route()` function, which we’ll take a look at further down, passing the router, the incoming request, and the application state to it.

After that, there is just some `hyper` boilerplate left, which tells the server which port to start on and actually starts it.

The final part to look at is the `route()` function.

```rust
async fn route(
    router: Arc<Router>,
    req: Request<hyper::Body>,
    app_state: AppState,
) -> Result<Response, Error> {
    let found_handler = router.route(req.uri().path(), req.method());
    let resp = found_handler
        .handler
        .invoke(Context::new(app_state, req, found_handler.params))
        .await;
    Ok(resp)
}
```

This is where we go from an incoming request to a handler function. We use the router’s route function with the request path and method to get to a `RouterMatch`. Then, we call `.invoke()` on the returned handler, giving it a new `Context` object containing the application state, the request, and the `params` from the `RouterMatch`. We await the handler future and return the response.

This is a simplistic route function, but you could imagine adding a `CORS` middleware, or really any kind of middleware, here - such as logging, for example. You could also do authorization handling here, checking incoming `Authorization` headers and making sure the caller has access to the requested resource.

That’s it! Let’s see if it works by executing `cargo run` and sending requests to it using `curl`.

```bash
curl http://localhost:8080/test
test called, state_thing was: state

curl http://localhost:8080/params/1234
param called, param was: 1234

curl -X POST http://localhost:8080/send -d '{"name": "chip", "active": true }'
send called with name: chip and active: true

curl -X POST http://localhost:8080/send -d '{"name": fsdfds, "active": true }'
HTTP/1.1 400 Bad Request
could not parse JSON: expected ident at line 1 column 11
```

Very nice! You can find the full example code at [GitHub](https://github.com/zupzup/rust-minimal-web-service-hyper).

## Conclusion

In my opinion, simplicity is a core value for building web services and software in general. The bigger a codebase and the more software packages it depends on, the harder it is to deal with the inherent and incidental complexity of that system. This can lead to degraded performance and subtle bugs you’ve never seen and don’t understand. In many cases, it can lead you to try to do something specific with a tool optimized for generic use.

Finding a sweet spot will require experimentation and courage. It all depends on the level of complexity you’re willing to accept for yourself (or your team) and the project you’re trying to implement.

You don’t have to build everything yourself. In fact, for some parts — especially security-critical things such as crypto libraries — you should always use battle-tested solutions.

Using small, lightweight libraries and a bit of self-written code to compose a system that can be considered minimal (or close to it) can help improve performance, maintainability, and code quality. I highly recommend this approach and would love to see more developers employ it.

