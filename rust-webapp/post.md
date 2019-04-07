I've been playing around with Rust quite a bit for the last several months and I have been loving it.

There is definitely a rather steep learning curve compared to learning other languages, but when it clicks and something compiles, it tends to work well in my experience so far, which is very rewarding.

One thing I like to do when learning a new language is to do something I'm conceptually familiar with, to see how it looks in the new language. This is often either a small CLI tool or a web application. In this case, we'll take a look at a basic web application in Rust.

There are several ways to go about this in Rust - there are several frameworks with different strengths. For this example, I decided to use [Actix](https://actix.rs/), because it seems mature and well documented/maintained.

In general, Rust is definitely not at the stage Java, node.js, Ruby or Go are when it comes to the Web ecosystem, but there is steady progress, which is documented [here](http://www.arewewebyet.org/).

Keep in mind that I'm both new to Rust and very new to Actix, so while what I threw together here works and does what I want it to do, it might not be the best/most optimal way of doing things. ;)

The web application we'll look at in this post exposes a small REST API, which proxies the incoming requests to another API.

In this example, I'll be using the [Timeular Public API](https://developers.timeular.com/public-api/) as a proxy target, because I'm very familiar with it. Anyone can create a free account and create API credentials, but the concepts of this post can be applied to any other web API as well.

The API we'll create is the following:

* `GET /rest/v1/activities` - returns all activities
* `POST /rest/v1/activities` - creates an activity
* `GET /rest/v1/{activity_id}` - returns an activity with the given id
* `PATCH /rest/v1/{activity_id}` - updates the activity with the given id
* `DELETE /rest/v1/{activity_id}` - deletes the activity with the given id

We'll also take a look at how to containerize the application using `docker`.

The `actix` framework handles requests asynchronously, but in this case the handlers themselves are not implemented asynchronously, for example using futures. I decided to keep it simple at first, but there will be a follow-up post, in which I will convert the handlers to an asynchronous model as well.

Alright, Let's go. :)

## Setup and Configuration 

In order to configure the service, in this case to set the `Timeular` credentials, we use the [envconfig](https://github.com/greyblake/envconfig-rs) crate, which basically converts given environment variables to a defined Rust struct.

In our case the configuration for it looks like this:

```rust
#[derive(Envconfig)]
pub struct Config {
    #[envconfig(from = "API_KEY", default = "")]
    pub api_key: String,

    #[envconfig(from = "API_SECRET", default = "")]
    pub api_secret: String,
}
```

And then, in `main`, we call:

```rust
let config = match Config::init() {
    Ok(v) => v,
    Err(e) => panic!("Could not read config from environment: {}", e),
};
```

For `logging` we use the [slog](https://github.com/slog-rs/slog) crate. I won't go into any details on logging, but there are several ways to set it up. The way we will use in our application is this:

```rust
pub fn setup_logging() -> slog::Logger {
    let decorator = slog_term::TermDecorator::new().build();
    let drain = slog_term::CompactFormat::new(decorator).build().fuse();
    let drain = slog_async::Async::new(drain).build().fuse();
    slog::Logger::root(drain, o!())
}
```

Which is taken directly from the documentation and creates an efficient, asynchronous way of logging.

Also, in order to make authenticated requests to the `Timeular` API, we need to log in first. Our application does this before starting up the web server and then passes the returned `JSON Web Token` to the handlers.

The login works like this:

```rust
let jwt = match external::get_jwt(&api_key, &api_secret) {
    Ok(v) => v,
    Err(e) => panic!("Could not get the JWT: {}", e),
};
```

Further down the post, we'll look at the actual `get_jwt` method in the `external` package, which does the `login` request, but for now, let's just say this works and we have a valid `JWT`.

Alright, those were the essentials - now we come to the actual actix web server.

Actix has the concept of `State`, which can be shared across handlers. This state object will be copied for every handler and we will put our `logging` handle and our `JWT` in there:

```rust
App::with_state(AppState {
    jwt: jwt.to_string(),
    log: log.clone(),
})
```

Then, we define our routes for the endpoints specified in the beginning of the post:

```rust
server::new(move || {
    App::with_state(AppState {
        jwt: jwt.to_string(),
        log: log.clone(),
    })
    .scope("/rest/v1", |v1_scope| {
        v1_scope.nested("/activities", |activities_scope| {
            activities_scope
                .resource("", |r| {
                    r.method(http::Method::GET).f(handlers::get_activities);
                    r.method(http::Method::POST)
                        .with_config(handlers::create_activity, |cfg| {
                            (cfg.0).1.error_handler(handlers::json_error_handler);
                        })
                })
                .resource("/{activity_id}", |r| {
                    r.method(http::Method::GET).with(handlers::get_activity);
                    r.method(http::Method::DELETE)
                        .with(handlers::delete_activity);
                    r.method(http::Method::PATCH)
                        .with_config(handlers::edit_activity, |cfg| {
                            (cfg.0).1.error_handler(handlers::json_error_handler);
                        });
                })
        })
    })
    .resource("/health", |r| {
        r.method(http::Method::GET).f(handlers::health)
    })
    .finish()
})
.bind("0.0.0.0:8080")
.unwrap()
.run();
```

So this snippet of code creates a new actix web server on port `8080`. First, we add our `state` object and then we define a `/rest/v1` scope, under which all defined routes will reside.

The next sub routes are `/` and `/{activity_id}`, which splits up the handlers between endpoints dealing with a specified `activity` and endpoints which don't.

Now that we defined this structure, the actual endpoints are rather simple, they are defined using the HTTP method such as `http::Method::GET` and then a handler function is provided, e.g.: `handlers::get_activities`.

For the `POST` and `PATCH` handlers, we also add a custom error handler called `handlers::json_error_handler`, which we will take a look at later on. This error handler basically deals with JSON parsing errors gracefully.

Also, we create a simple `/health` endpoint, which just returns `OK` if the service is up.

## External API calls

Before we get into the actual handlers, we will look at the `external` package, which is basically just a very basic HTTP client for the Timeular API.

For making HTTP requests, we'll use the [reqwest](https://github.com/seanmonstar/reqwest) crate, which has a very simple API.

We won't look at all the requests, as they're rather similar, but the full code is on GitHub (link at the bottom).

The basic API of the `external` package is to have one function for each individual request, such as `get_activities`:

```rust
pub fn get_activities(jwt: &str) -> Result<ActivitiesResponse, Error> {
    let activities_path = format!("{}/activities", BASE_URL);
    let result = get(&activities_path, jwt)?;
    serde_json::from_str(&result).map_err(|e| format_err!("could not parse json, reason: {}", e))
}
```

These functions are actually rather simple. First, we define the path to request and then pass it and our `JSON Web Token` to a helper method for the HTTP method we're using, which looks like this:

```rust
fn get(path: &str, jwt: &str) -> Result<String, Error> {
    let client = reqwest::Client::new();
    let res = client
        .get(path)
        .header("Authorization", format!("Bearer {}", jwt))
        .send()
        .context("error during get request")?;
    parse_result(res)
}
```

Here, we create a new `reqwest` client, configure a request and send it. At the end, we parse the result as follows:

```rust
fn parse_result(mut res: Response) -> Result<String, Error> {
    let mut buf: Vec<u8> = vec![];
    if res.status().is_success() {
        res.copy_to(&mut buf)
            .context("could not copy response into buffer")?;
    } else {
        return Err(format_err!("request error: {}", res.status()));
    }
    let result = std::str::from_utf8(&buf)?;
    Ok(result.to_string())
}
```

This helper method basically just parses the given response to a String.

As you could see above in `get_activities`, we then parse this String to a `serde` data object called `ActivitiesResponse`, which is simply a list of `Activities:`


```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct ActivitiesResponse {
    pub activities: Vec<ActivityResponse>,
}

#[derive(Serialize, Deserialize, Debug, Clone)]
#[serde(rename_all = "camelCase")]
pub struct ActivityResponse {
    pub id: String,
    pub name: String,
    pub color: String,
    pub integration: String,
    pub device_side: Option<Number>,
}
```

For each of these operations, the errors are handled and returned in the `Result`.

To also show an example of a request, which sends something to the server, we'll also look at the above mentioned `get_jwt` method:

```rust
pub fn get_jwt(api_key: &str, api_secret: &str) -> Result<String, Error> {
    let mut body = HashMap::new();
    body.insert("apiKey", api_key);
    body.insert("apiSecret", api_secret);
    let jwt_path = format!("{}/developer/sign-in", BASE_URL);
    let result = post(&jwt_path, &body, "")?;
    let json: Result<SignInResponse, Error> = serde_json::from_str(&result)
        .map_err(|e| format_err!("could not parse json, reason: {}", e));
    Ok(json.unwrap().token)
}
```

Basically, the incoming payload is simply converted to a `HashMap` and then passed to `reqwest` as a body, the rest is very similar to the example above.

## Handlers

Having just looked at the external API, the handlers are actually rather simple. We'll look at both a simple `GET` request as well as a mutating `POST` request.

Again, we'll look at the most simple handler function called `get_activities`:

```rust
pub fn get_activities(
    req: &HttpRequest<AppState>,
) -> Result<Json<ActivitiesResponse>, AnalyzerError> {
    let jwt = &req.state().jwt;
    let log = &req.state().log;
    external::get_activities(jwt)
        .map_err(|e| {
            error!(log, "Get Activities ExternalServiceError {}", e);
            AnalyzerError::ExternalServiceError
        })
        .map(Json)
}
```

The basic structure of all the handlers is, to get the `logging` handle and the `JWT` from the server `state` and to call the relevant `external` method, which then calls the Timeular API.

Additionally, if an error happens, that error is converted to a custom error type. To be able to do this, we need to implement the `error:ResponseError` trait of Actix:

```rust
impl error::ResponseError for AnalyzerError {
    fn error_response(&self) -> HttpResponse {
        match *self {
            AnalyzerError::ExternalServiceError => HttpResponse::InternalServerError()
                .content_type("text/plain")
                .body("external service error"),
            AnalyzerError::ActivityNotFoundError => HttpResponse::NotFound()
                .content_type("text/plain")
                .body("activity not found"),
        }
    }
}
```

Handlers in actix have to return something implementing actix-web's `Responder` trait, which is quite flexible. For example, the `/health` endpoint simply returns a string:

```rust
pub fn health(_: &HttpRequest<AppState>) -> impl Responder {
    "OK".to_string()
}
```

The JSON endpoints however, return a `Result<Json<DomainObject>, AnalyzerError>`, where the `DomainObject` is the data object returned from the `external` API call, which is wrapped with actix-web's `Json` type, automatically returning nice JSON to the caller.

Alright, now let's look at an endpoint, which sends something to the API, `create_activity`:

```rust
pub fn create_activity(
    (req, activity): (HttpRequest<AppState>, Json<ActivityRequest>),
) -> Result<Json<ActivityResponse>, AnalyzerError> {
    let jwt = &req.state().jwt;
    let log = &req.state().log;
    info!(log, "creating activity {:?}", activity);
    external::create_activity(&activity, jwt)
        .map_err(|e| {
            error!(log, "Create Activity ExternalServiceError {}", e);
            AnalyzerError::ExternalServiceError
        })
        .map(Json)
}
```

As you can see, it's relatively similar, except for the `Json<ActivityRequest>`, which is passed to the handler:

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct ActivityRequest {
    pub name: String,
    pub color: String,
    pub integration: String,
}
```

Passing this data object to the handler works basically automatically in actix. With `Json<ActivityRequest>` being in the signature of the handler, actix expects there to be a JSON payload, which needs to be parseable to the given data object.

The rest of the handler is the same as the above mentioned `GET` endpoint.


The `json_error_handler` was mentioned above, which deals with errors during parsing an incoming JSON payload. In order to do this, a custom error handler needs to be implemented like this:

```rust
pub fn json_error_handler(err: error::JsonPayloadError, _: &HttpRequest<AppState>) -> Error {
    error::InternalError::from_response(
        "",
        HttpResponse::BadRequest()
            .content_type("application/json")
            .body(format!(r#"{{"error":"json error: {}"}}"#, err)),
    )
    .into()
}
```

Here, we create an error handler, which is used when a `JsonPayloadError` happens in actix. If this happens, we return a HTTP 400 error, telling the user that the JSON was not correct. This custom error handler is used for all endpoints, which receive a JSON payload.

That's it. :)

Now we can just start up the server using `cargo run` with our credentials as environment variables and interact with our Timeular activities through this rust web application.

The full code can be found [here](https://github.com/zupzup/rust-web-example).

## Dockerizing the Application

In order to be able to use a web application such as this in a production setting, it would be nice to containerize it using `docker`. I used this `Dockerfile` to achieve this:

```bash
FROM ekidd/rust-musl-builder:nightly as builder
ADD . ./
RUN sudo chown -R rust:rust /home/rust
RUN cargo build --release

FROM alpine:latest
RUN apk --no-cache add ca-certificates
COPY --from=builder /home/rust/src/target/x86_64-unknown-linux-musl/release/analyzer /usr/local/bin/analyzer
EXPOSE 8080
CMD ["/usr/local/bin/analyzer"]
```

First, we build the application using the `rust-musl-builder` image and in a second step, we move the created linux binary to a plain alpine image and run it there.

This approach has the benefit, that the resulting image is very small and doesn't include the whole Rust toolchain.

## Conclusion

I'm still very much at the beginning of my journey with Rust and after the initial bit of frustration getting used to the borrow checker and the strict compiler, it has been a lot of fun and very rewarding.

Regarding the possibility to write web applications using Rust - I'd say the necessary libraries and tools are there, but it is of course not on the level of other languages yet in this regard.

However, there seems to be a lot of progress in this area, so I'm sure Rust will catch up over the next time, plus there's a lot of other fun stuff you can do with rust besides building web applications. :)

#### Resources

* [Actix](https://actix.rs/)
* [Code Example](https://github.com/zupzup/rust-web-example)
* [Timeular](https://timeular.com/)

