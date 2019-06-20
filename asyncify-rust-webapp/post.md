In a [previous post](https://zupzup.org/rust-webapp/) I showed an example Rust web app using [Actix 0.7](https://actix.rs/) and synchronous handlers.

This time, we'll move that simple app from Actix 0.7 to Actix 1.x and we'll completely asyncify it. I would suggest you read the above-mentioned post, or at least glance over the [code example](https://github.com/zupzup/rust-web-example) in order to follow this post and the changes we'll make.

The first step will be to asyncify the handlers in the old Actix 0.7 application and then we'll move the whole app to Actix 1.0. The point of this is to first see how easy (or hard) it is to make the app fully non-blocking.

Let's go!

## Moving to Async Handlers

Alright, so this section will show how we move from synchronously handling requests in actix to asynchronously handling them. The issue in the previous implementation was, that while actix handles incoming requests asynchronously by default, we used the synchronous version of the [reqwest](https://github.com/seanmonstar/reqwest) library, so we had something blocking on every request.

In this example, we will change this by using the `async` module of `reqwest`.

Unfortunately, it's quite hard to do this one step at a time and keep everything compiling, as we will pass `Future`s through the whole stack instead of `Result`s. However, it's not that much code which needs to be changed and we'll look at two handlers (one `POST` and one `GET`) to see the differences from the handler to the external HTTP call.

We'll start with the HTTP call first and work our way back to the handler. The first thing we need to do is using the async version of `reqwest`:

```rust
use reqwest::r#async::{Client, Response};
use futures::Future;
```

We need the `r#async` qualifier, because `async` is now a reserved word in rust.

Ok, let's start with the `GET` request which fetches activities. It's the simplest case we have and we need to transform it to return an `impl Future` instead of a `Result`, while still propagating the error as before:

```rust
fn get(path: &str, jwt: &str) -> impl Future<Item = Response, Error = reqwest::Error> {
    let client = Client::new();
    client
        .get(path)
        .header("Authorization", format!("Bearer {}", jwt))
        .send()
        .and_then(|res: Response| futures::future::ok(res))
        .map_err(|err| {
            println!("Error during get request: {}", err);
            err
        })
}

pub fn get_activities(jwt: &str) -> impl Future<Item = ActivitiesResponse, Error = reqwest::Error> {
    let activities_path = format!("{}/activities", BASE_URL);
    get(&activities_path, jwt).and_then(|mut res: Response| res.json::<ActivitiesResponse>())
}
```

As you can see, not much has changed here - the `Result<ActivitiesResponse, reqwest::Error>` is now an `impl Future<Item = ActivitiesResponse,, Error = request::Error>` and we need to use `and_then` and `map_err` to return the correct types.

On the side of `reqwest`, just using the `async` Client, wrapping the response in a `futures::future::ok` and mapping the error is enough to get the job done.

Alright, that wasn't so bad. Let's look at a `POST` request next, where we have to handle a payload as well.

```rust
fn post(
    path: &str,
    body: &HashMap<&str, &str>,
    jwt: &str,
) -> impl Future<Item = Response, Error = reqwest::Error> {
    let client = Client::new();
    client
        .post(path)
        .json(&body)
        .header("Authorization", format!("Bearer {}", jwt))
        .send()
        .and_then(|res: Response| futures::future::ok(res))
        .map_err(|err| {
            println!("Error during post request: {}", err);
            err
        })
}

pub fn create_activity(
    activity: &ActivityRequest,
    jwt: &str,
) -> impl Future<Item = ActivityResponse, Error = Error> {
    let mut body: HashMap<&str, &str> = HashMap::new();
    body.insert("name", &activity.name);
    body.insert("color", &activity.color);
    body.insert("integration", &activity.integration);
    let path = format!("{}/activities", BASE_URL);
    post(&path, &body, jwt)
        .and_then(|mut res: Response| res.json::<ActivityResponse>())
        .map_err(|e| format_err!("error creating activity, reason: {}", e))
}
```

The change is really the same as in the `GET` example - in this case we convert the error to a `failure::Error`, but other than that it's the same. The code for preparing the payload and adding it to the request didn't change either, so we're safe here.

We can repeat this exercise for all the handlers and `reqwest` wrapper functions - it's the same process for all.

After changing all the fetching functions, actually fetching data returns a `Future` in each case. Next, we'll have to asyncify the actix handlers which call the methods we updated above. The process is basically the same as for the HTTP calls. Again, we'll start with the handler for fetching activities:

```rust
pub fn get_activities(
    state: State<AppState>,
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &state.jwt;
    let log = state.log.clone();
    external::get_activities(jwt)
        .map_err(move |e| {
            error!(log, "Get Activities ExternalServiceError: {}", e);
            AnalyzerError::ExternalServiceError
        })
        .and_then(|data| json_ok(&data))
}
```

Again, we simply change the return type from a `Result` to an `impl Future`. This time however, instead of using the `Json` method to convert and return a data object from the function, we actually return our own `HttpResponse` using the `json_ok` helper method.

We use this `json_ok` function for wrapping up our response into a JSON payload. It also sets the content type and the body into a `200 OK` HTTP response:

```rust
fn json_ok<T: ?Sized>(data: &T) -> Result<HttpResponse, AnalyzerError>
where
    T: serde::ser::Serialize,
{
    Ok(HttpResponse::Ok()
        .content_type("application/json")
        .body(serde_json::to_string(&data).unwrap())
        .into())
}
```

Alright, as we have done above with our HTTP call methods, we'll also look at the `POST` handler. Spoiler alert: There's not much difference to the other changes.

```rust
pub fn create_activity(
    (req, activity): (HttpRequest<AppState>, Json<ActivityRequest>),
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &req.state().jwt;
    let log = req.state().log.clone();
    info!(log, "creating activity {:?}", activity);
    external::create_activity(&activity, jwt)
        .map_err(move |e| {
            error!(log, "Create Activity ExternalServiceError {}", e);
            AnalyzerError::ExternalServiceError
        })
        .and_then(|data| json_ok(&data))
}
```

Ok, now the last remaining thing is to hook up the `async` handlers using the correct routing function in `main.rs`:

```rust
.scope("/rest/v1", |v1_scope| {
	v1_scope.nested("/activities", |activities_scope| {
		activities_scope
			.resource("", |r| {
				r.method(http::Method::GET)
					.with_async(handlers::get_activities);
				r.method(http::Method::POST).with_async_config(
					handlers::create_activity,
					|cfg| {
						(cfg.0).1.error_handler(handlers::json_error_handler);
					},
				)
			})
			.resource("/{activity_id}", |r| {
				r.method(http::Method::GET)
					.with_async(handlers::get_activity);
				r.method(http::Method::DELETE)
					.with_async(handlers::delete_activity);
				r.method(http::Method::PATCH).with_async_config(
					handlers::edit_activity,
					|cfg| {
						(cfg.0).1.error_handler(handlers::json_error_handler);
					},
				);
			})
	})
})
```

The differences here are just the usage of `with_async` and `with_async_config`, which are the methods used for future-based handlers.

That's it - now our handlers are fully non-blocking. :)

Next up, we'll migrate the whole app to Actix 1.0

## Moving to Actix 1.0

We'll use the Actix [Migration Guide](https://github.com/actix/actix-web/blob/master/MIGRATION.md) to see what we need to change to move to 1.0. The first and most important step is of course, to update the dependency and see what breaks.

```yml
actix-web = "1.0.0"
```

The biggest difference for our app is the new resource registration API, which uses the new `service()` method, which together with `web::scope` and `web::resource` is used to structure the routes.

Also, `actix_web::server` is replaced by `HttpServer`, which returns a `Result` when calling `.run()` on it.

The new Actix 1.0 compatible route configuration looks like this:

```rust
match HttpServer::new(move || {
	App::new()
		.data(AppState {
			jwt: jwt.to_string(),
			log: log.clone(),
		})
		.service(web::scope("/rest/v1").service(
			web::scope("/activities").service(web::resource("")
				.route(web::get().to_async(handlers::get_activities))
				.route(web::post().to_async(handlers::create_activity)),
			)
			.service(web::resource("/{activity_id}")
				.route(web::get().to_async(handlers::get_activity))
				.route(web::delete().to_async(handlers::delete_activity))
				.route(web::patch().to_async(handlers::edit_activity)),
		)))
		.service(web::resource("/health").route(web::get().to(handlers::health)))
})
.bind("0.0.0.0:8080")
.unwrap()
.run()
{
	Ok(_) => info!(runner_log, "Server Stopped!"),
	Err(e) => error!(runner_log, "Error running the server: {}", e),
};
```

Another change is how `AppState` is passed to the `App`. This now happens by using the `.data()` function instead of `with_state()`. Other than that, custom error handling is now simpler and it seems that the whole concept of having to configure custom error handling separately using `with_async_config` as such has gone away. Nice.

This means, that our handler config is actually shorter and a lot more readable.

The next big change is, that all handler functions must use extractors now, so there is only `.to()` and `.to_async()`, no `.f()` etc. anymore.

However, this means that all of our handler functions need to change, because they all used `HttpRequest<AppState>` and extracted the data manually from there.

So

```rust
pub fn get_activities(
    state: State<AppState>,
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &state.jwt;
    let log = state.log.clone();
	...
```

becomes

```rust
pub fn get_activities(
    data: Data<AppState>,
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &data.jwt;
    let log = data.log.clone();
	...
```

And

```rust
pub fn get_activity(
    (req, activity_id): (HttpRequest<AppState>, Path<String>),
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &req.state().jwt;
    let log = req.state().log.clone();
	...
```

becomes

```rust
pub fn get_activity(
    data: Data<AppState>,
    activity_id: Path<String>,
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &data.jwt;
    let log = data.log.clone();
	...
```

And finally

```rust
pub fn create_activity(
    (req, activity): (HttpRequest<AppState>, Json<ActivityRequest>),
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &req.state().jwt;
    let log = req.state().log.clone();
	...
```

becomes

```rust
pub fn create_activity(
    data: Data<AppState>,
    activity: Json<ActivityRequest>,
) -> impl Future<Item = HttpResponse, Error = AnalyzerError> {
    let jwt = &data.jwt;
    let log = data.log.clone();
	...
```

I think you get the idea... :)

Another change was, that `Path`, `Json` and `Data` were moved to the `actix_web::web` package, so we need to use:

```rust
use actix_web::web::{Data, Json, Path}; 
```

We can also remove our `json_error_handler` method, as we don't need to configure the custom error handler in this way anymore.

Perfect - we now have a non-blocking Actix 1.0 Web App, having started from a partly blocking Actix 0.7 App. Exactly what we set out to do.:)

The full example code can be found [here](https://github.com/zupzup/rust-async-web-example)

## Conclusion

Actix 1.0 is an important milestone and from what I gather they made the framework a lot more approachable and improved the ergonomics quite a bit as well.

Moving my simple app to async and to 1.0 was a lot easier than I expected. Frankly, I have to question my initial approach of not making it async in the first place to keep it simple, as really there isn't much more complexity involved, both with Actix 0.7 and especially with 1.0.

I hope this was a useful guide and I'm looking forward to doing more Actix-related things in the future. :)

#### Resources

* [Actix](https://actix.rs/)
* [Actix Migration Guide](https://github.com/actix/actix-web/blob/master/MIGRATION.md)
* [Code Example](https://github.com/zupzup/rust-async-web-example)
* [Reqwest](https://github.com/seanmonstar/reqwest)
