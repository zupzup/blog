*This post was originally posted on the [LogRocket](https://blog.logrocket.com/timezone-handling-in-rust-with-chrono-tz/) blog on 26.10.2020 and was cross-posted here by the author.*

When dealing with dates and times in a modern web application - especially one that serves users from all around the world - handling time zones can become a concern.

Depending on the use case, time zones can be rather tricky. A quick look at the [IANA Time Zone Database](https://www.iana.org/time-zones) and its release history, and you’ll notice another problem: time zones can change. In just the last five years, there have been time zone changes in North Korea, Russia, Haiti, Chile, and more. Some of these changes involve adding or removing daylight saving time, others are based on different factors.

If you need to use exact time zones in your application, it’s important to keep them up to date. Let’s say, for example, that you want to send an email or notification to all users at a certain point in time in their respective time zone. Depending on how much notice countries give and how fast a database change can be propagated, there might be a period during which a time zone is plain wrong in your app.

In this tutorial, we’ll show you how to deal with dates and time zones in Rust using the [Chrono](https://github.com/chronotope/chrono) and [Chrono-TZ](https://github.com/chronotope/chrono-tz) libraries. Chrono is a go-to crate for handling dates and time in Rust and Chrono-TZ is an extension for dealing with time zones.

We’ll build a simple web service that enables users to add dates in [RFC3339](https://www.ietf.org/rfc/rfc3339.txt) format (“1996-12-19T16:39:57+02:00”), which includes the time zone. These dates are then saved in-memory and users can fetch and convert them to a time-zone of their choosing.

In building this app, we’ll demonstrate how to parse and format dates in Rust, as well as how to parse and convert time zones.

## Setup

To follow along, you’ll need a recent Rust installation (1.39+) and a tool to send HTTP requests, such as cURL.

First, create a new Rust project.

```bash
cargo new rust-timezone-example
cd rust-timezone-example
```

Next, edit the `Cargo.toml` file and add the requisite dependencies.

```toml
[dependencies]
tokio = { version = "0.2", features = ["macros", "rt-threaded", "sync"] }
warp = "0.2"
chrono = { version = "0.4", features = ["serde"] }
chrono-tz = "0.5"
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

The service will be written in warp using tokio underneath. We’ll use `serde` to deserialize the incoming request payload.

## Data storage

We’ll use an in-memory data storage mechanism in the form of a shared vector of dates.

```rust
use chrono::prelude::*;
use tokio::sync::RwLock;
use std::sync::Arc;

type Dates = Arc<RwLock<Vec<DateTime<Utc>>>>;
```

The `Dates` type describes our data store of dates. It has a somewhat scary list of types in front of it, but it’s actually nothing fancy.

Let’s go from the inside out. We’ll save dates in `UTC`. That way, we don’t have to keep track of the incoming date’s time zone; we can simply convert between them. The `DateTime` and `Utc` types are from Chrono’s prelude.

We want to save a list of dates, so these dates are put into a `Vec`. Because we’ll use this as data storage for an asynchronous web service, we might read and write from this data storage from different threads, potentially at the same time. For this reason, we’ll put an async `RwLock` from the tokio runtime around it, so anyone who wants to access it has to acquire a lock. An `RwLock` grants multiple readers a lock, but only one write-lock can be active at a time. This guarantees that we won’t have any data races.

Finally, we put the whole thing inside an `Arc` - an atomic reference counted smart pointer — so we can share it across threads in a memory-safe way.

## Web server

The next step is to set up a basic web service and wire it up with this makeshift data store. Let’s start with the warp server in `main`.

```rust
use warp::{http::StatusCode, reply, Filter, Rejection, Reply};
use std::convert::Infallible;

#[tokio::main]
async fn main() {
    let dates: Dates = Arc::new(RwLock::new(Vec::new()));

    let create_route = warp::path("create")
        .and(warp::post())
        .and(with_dates(dates.clone()))
        .and(warp::body::json())
        .and_then(create_handler);
    let fetch_route = warp::path("fetch")
        .and(warp::get())
        .and(warp::path::param())
        .and(with_dates(dates.clone()))
        .and_then(fetch_handler);

    println!("Server started at localhost:8080");
    warp::serve(create_route.or(fetch_route))
        .run(([0, 0, 0, 0], 8080))
        .await;
}

fn with_dates(dates: Dates) -> impl Filter<Extract = (Dates,), Error = Infallible> + Clone {
    warp::any().map(move || dates.clone())
}
```

We create two routes, one for `POST /create` and one for `GET /fetch/$timezone`. In both cases, we pass in our `dates` data storage, which we initialize at the beginning of `main`, to the handler via a warp filter called `with_dates`, which just copies an atomic reference into the handlers.

The `create` endpoint also takes a payload of type `DateTimeRequest`.

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct DateTimeRequest {
    date_time: String,
}
```

Finally, we need to define two placeholder handler functions for these routes.

```rust
type Result<T> = std::result::Result<T, Rejection>;

async fn create_handler(dates: Dates, body: DateTimeRequest) -> Result<impl Reply> {
    Ok("")
}

async fn fetch_handler(time_zone: String, dates: Dates) -> Result<impl Reply> {
    Ok("")
}
```

These don’t do anything interesting yet, but that’s what we’ll tackle next.

## Creating dates

For the endpoint used to create dates, the first step is to parse the incoming `body.date_time` string into a valid date. Then, if that works out, we’ll convert the date to UTC and push it to the data store, returning a success message.

Let’s look at this translated into Rust code.

```rust
async fn create_handler(dates: Dates, body: DateTimeRequest) -> Result<impl Reply> {
    let dt: DateTime<FixedOffset> = match DateTime::parse_from_rfc3339(&body.date_time) {
        Ok(v) => v,
        Err(e) => {
            return Ok(reply::with_status(
                format!("could not parse date: {}", e),
                StatusCode::BAD_REQUEST,
            ))
        }
    };

    dates.write().await.push(dt.with_timezone(&Utc));

    Ok(reply::with_status(
        format!("Added date with timezone: {} as UTC", dt.timezone()),
        StatusCode::OK,
    ))
}
```

We use Chrono’s `DateTime::parse_from_rfc3339` function to parse the date. What comes out here is a `DateTime<FixedOffset>`. The offset describes the time zone. There are more ways to parse dates with arbitrary, user-defined formats in Chrono as well.

If this fails, we return an error. Otherwise, we get a write lock to our data store using `dates.write().await` and push the date, converted to `UTC`, to the vector of dates.

Finally, we return a success message to the user, showing them the parsed time zone from the incoming date.

## Fetching dates

To fetch all saved dates, users can use the `GET /fetch/$timezone` endpoint, where `$timezone` might be something like `UTC`, `GMT`, `Africa/Algiers`, or any of the [many time zones](https://docs.rs/chrono-tz/0.5.3/chrono_tz/enum.Tz.html) Chrono-TZ and the IANA support.

For cases such as `Europe/Vienna`, the caller needs to url encode the `/` to `%2F` when calling the endpoint, and we’d have to url decode it again. For this basic example, we’ll do a simple replacement to handle this case.

The next step is to parse the incoming time zone and, if it’s a valid time zone, return all saved dates converted to this time zone to the user.

Let’s look at one possible solution:

```rust
async fn fetch_handler(time_zone: String, dates: Dates) -> Result<impl Reply> {
    let parsed_time_zone = time_zone.replace("%2F", "/");
    let tz: Tz = match parsed_time_zone.parse() {
        Ok(v) => v,
        Err(e) => {
            return Ok(reply::with_status(
                format!("could not parse timezone: {}", e),
                StatusCode::BAD_REQUEST,
            ))
        }
    };

    Ok(
        match serde_json::to_string(
            &dates
                .read()
                .await
                .iter()
                .map(|t: &DateTime<Utc>| t.with_timezone(&tz).to_rfc3339())
                .collect::<Vec<_>>(),
        ) {
            Ok(v) => reply::with_status(v, StatusCode::OK),
            Err(e) => {
                return Ok(reply::with_status(
                    format!("could not serialize json: {}", e),
                    StatusCode::INTERNAL_SERVER_ERROR,
                ))
            }
        },
    )
}
```

After our replacement maneuver, we use the `.parse()` function to parse the incoming string to a `chrono_tz::Tz`. If this fails, we return an error.

Otherwise, the next step is to acquire a read lock on the data store using `dates.read().await` and then iterate the vector, mapping the UTC dates within dates converted to the given timezone using the `.with_timezone` helper.

Finally, we convert the dates to an RFC3339 date string to be consistent with the input. The resulting vector is then serialized to JSON and returned to the user.

That’s it! Let’s see if it works as we’d expect.

First, run the server using cargo run, then create a date using cURL.

```bash
curl -X POST http://localhost:8080/create -d '{"date_time": "1996-12-19T16:39:57+02:00"}' -H "content-type: application/json"
Added date with timezone: +02:00 as UTC
```

So far, so good. Let’s fetch it in UTC first.

```bash
curl http://localhost:8080/fetch/UTC
["1996-12-19T14:39:57+01:00"]
```

And then in another time zone: Africa/Algiers, which is UTC+01:00.

```bash
curl http://localhost:8080/fetch/Africa%2FAlgiers
["1996-12-19T15:39:57+00:00"]
```

Very nice! We saved a date in UTC+02:00, then fetched it with UTC, which was two hours less, and `Africa/Algiers`, which is UTC+01:00 (one hour less).

You can have fun trying this out with different time zone combinations. Check out the full example code on [GitHub](https://github.com/zupzup/rust-timezones-chrono-tz).

## Conclusion

Time and date handling, especially with time zones, are tricky business. Fortunately, Rust’s ecosystem provides us with all the tools we need.

The Chrono libraries are robust, complete and widely used, and they have great APIs. Crates such as these make the technical aspect of handling time zones a nonissue. Dealing with actual business problems around dates and time is difficult enough without having to fight your way through strange APIs, inconsistencies, and bugs.

