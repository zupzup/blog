*This post was originally posted on the [LogRocket](https://blog.logrocket.com/using-redis-in-a-rust-web-service/) blog on 29.06.2020 and was cross-posted here by the author.*

Redis has been a staple of the web ecosystem for years. It’s often used for caching, as a message broker, or simply as a database.

In this guide, we’ll demonstrate how to use [Redis](https://redis.io/) inside a Rust web application.

We’ll build the application using the fantastic [warp](https://github.com/seanmonstar/warp) web framework. The techniques in this tutorial will work very similarly with other Rust web stacks.

We’ll explore three approaches to using Redis:

* Directly, using one asynchronous connection per request
* Using a synchronous connection pool
* Using an asynchronous connection pool

As a client library for Redis, [redis-rs](https://github.com/mitsuhiko/redis-rs) is the most stable and widely used crate, so all variants use it as their basis.

For the synchronous pool, we’ll use the [r2d2](https://github.com/sfackler/r2d2)-based [r2d2-redis](https://github.com/sorccu/r2d2-redis). We’ll use [mobc](https://github.com/importcjj/mobc) for the asynchronous solution. There are plenty of other async connection pools, such as [deadpool](https://github.com/bikeshedder/deadpool) and [bb8](https://github.com/khuey/bb8), that all work in a similar way.

The application itself doesn’t do much. There is a handler for each of the different Redis connection approaches, which puts a hardcoded string into Redis with an expiration time of 60 seconds and then reads it out again.

Without further ado, let’s get started!

## Setup

First, let’s set up some shared types and define a module for each Redis connection approach:


```rust
mod direct;
mod mobc_pool;
mod r2d2_pool;

type WebResult<T> = std::result::Result<T, Rejection>;
type Result<T> = std::result::Result<T, Error>;

const REDIS_CON_STRING: &str = "redis://127.0.0.1/";
```

The two `Result` types are defined to save some typing and to represent internal `Errors` (`Result`) and external `Errors` (`WebResult`).

Next, define this internal `Error`-type and implement `Reject` for it so it can be used to return HTTP errors from handlers.

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("mobc error: {0}")]
    MobcError(#[from] MobcError),
    #[error("direct redis error: {0}")]
    DirectError(#[from] DirectError),
    #[error("r2d2 error: {0}")]
    R2D2Error(#[from] R2D2Error),
}

#[derive(Error, Debug)]
pub enum MobcError {
    #[error("could not get redis connection from pool : {0}")]
    RedisPoolError(mobc::Error<mobc_redis::redis::RedisError>),
    #[error("error parsing string from redis result: {0}")]
    RedisTypeError(mobc_redis::redis::RedisError),
    #[error("error executing redis command: {0}")]
    RedisCMDError(mobc_redis::redis::RedisError),
    #[error("error creating Redis client: {0}")]
    RedisClientError(mobc_redis::redis::RedisError),
}

#[derive(Error, Debug)]
pub enum R2D2Error {
    #[error("could not get redis connection from pool : {0}")]
    RedisPoolError(r2d2_redis::r2d2::Error),
    #[error("error parsing string from redis result: {0}")]
    RedisTypeError(r2d2_redis::redis::RedisError),
    #[error("error executing redis command: {0}")]
    RedisCMDError(r2d2_redis::redis::RedisError),
    #[error("error creating Redis client: {0}")]
    RedisClientError(r2d2_redis::redis::RedisError),
}

#[derive(Error, Debug)]
pub enum DirectError {
    #[error("error parsing string from redis result: {0}")]
    RedisTypeError(redis::RedisError),
    #[error("error executing redis command: {0}")]
    RedisCMDError(redis::RedisError),
    #[error("error creating Redis client: {0}")]
    RedisClientError(redis::RedisError),
}

impl warp::reject::Reject for Error {}
```

This is a lot of boilerplate for a couple of error types, but since the goal is to implement three ways to do the same thing, this is hard to avoid if we want to have nice errors.

The above defines the general error type and `Error` types for each of the Redis approaches we will implement. The errors themselves simply deal with connection, pool creation, and command execution errors. You’ll see them in action later.

## Using redis-rs directly (async)

Now it’s time to implement the first way to interact with Redis - using one connection per request. The idea is to create a new connection for every request that comes in. This approach is fine if there isn’t a lot of traffic, but won’t scale well with hundreds or thousands of concurrent requests.

redis-rs supports both a synchronous and an asynchronous API. The maintainers did a great job keeping the two APIs very similar. While we’ll focus on the asynchronous way to use the crate directly, the synchronous version would look almost exactly the same.

First, let’s establish a new connection.

```rust
use crate::{DirectError::*, Result};
use redis::{aio::Connection, AsyncCommands, FromRedisValue};

pub async fn get_con(client: redis::Client) -> Result<Connection> {
    client
        .get_async_connection()
        .await
        .map_err(|e| RedisClientError(e).into())
}
```

In the above snippet, a `redis::Client` is passed in and an async connection comes out, handling the error. We’ll look at the creation of the `redis::Client` later.

Now that it’s possible to open a connection, the next step is to create some helpers to enable setting values in Redis and get them out again. In this simple case, we’ll only focus on `String` values.

```rust
pub async fn set_str(
    con: &mut Connection,
    key: &str,
    value: &str,
    ttl_seconds: usize,
) -> Result<()> {
    con.set(key, value).await.map_err(RedisCMDError)?;
    if ttl_seconds > 0 {
        con.expire(key, ttl_seconds).await.map_err(RedisCMDError)?;
    }
    Ok(())
}

pub async fn get_str(con: &mut Connection, key: &str) -> Result<String> {
    let value = con.get(key).await.map_err(RedisCMDError)?;
    FromRedisValue::from_redis_value(&value).map_err(|e| RedisTypeError(e).into())
}
```

This part will be very similar between all three implementations since this is just the redis-rs API in action. The API closely mirrors Redis commands and the FromRedisValue trait is a convenient way to convert values into the expected data types.

So far, so good. Next up is a synchronous pool based on the widely used r2d2 crate.

## Using r2d2 (sync)

The [r2d2](https://github.com/sfackler/r2d2) crate was, to my knowledge, the first widely used connection pool, and it still enjoys widespread use. The Redis flavor for this base crate is [r2d2-redis](https://github.com/sorccu/r2d2-redis).

Since we’re using a connection pool now, the creation of this pool needs to be handled in the r2d2 module as well. Here’s the starting point:

```rust
pub type R2D2Pool = r2d2::Pool<RedisConnectionManager>;
pub type R2D2Con = r2d2::PooledConnection<RedisConnectionManager>;

const CACHE_POOL_MAX_OPEN: u32 = 16;
const CACHE_POOL_MIN_IDLE: u32 = 8;
const CACHE_POOL_TIMEOUT_SECONDS: u64 = 1;
const CACHE_POOL_EXPIRE_SECONDS: u64 = 60;

pub fn connect() -> Result<r2d2::Pool<RedisConnectionManager>> {
    let manager = RedisConnectionManager::new(REDIS_CON_STRING).map_err(RedisClientError)?;
    r2d2::Pool::builder()
        .max_size(CACHE_POOL_MAX_OPEN)
        .max_lifetime(Some(Duration::from_secs(CACHE_POOL_EXPIRE_SECONDS)))
        .min_idle(Some(CACHE_POOL_MIN_IDLE))
        .build(manager)
        .map_err(|e| RedisPoolError(e).into())
}
```
After defining some constants to configure the pool, such as open and idle connections, connection timeout and the lifetime of a connection, the pool itself is created using the `RedisConnectionManager`, which gets the redis connection string passed to it.

Don’t worry about the configuration values too much; they’re just here to show off that they are available. Setting them properly will depend on your use case, and most connection pools have some defaults that will work for basic applications.

We need a way to get a connection from the pool and the helpers to set and get from Redis again.

```rust
pub fn get_con(pool: &R2D2Pool) -> Result<R2D2Con> {
    pool.get_timeout(Duration::from_secs(CACHE_POOL_TIMEOUT_SECONDS))
        .map_err(|e| {
            eprintln!("error connecting to redis: {}", e);
            RedisPoolError(e).into()
        })
}

pub fn set_str(pool: &R2D2Pool, key: &str, value: &str, ttl_seconds: usize) -> Result<()> {
    let mut con = get_con(&pool)?;
    con.set(key, value).map_err(RedisCMDError)?;
    if ttl_seconds > 0 {
        con.expire(key, ttl_seconds).map_err(RedisCMDError)?;
    }
    Ok(())
}

pub fn get_str(pool: &R2D2Pool, key: &str) -> Result<String> {
    let mut con = get_con(&pool)?;
    let value = con.get(key).map_err(RedisCMDError)?;
    FromRedisValue::from_redis_value(&value).map_err(|e| RedisTypeError(e).into())
}
```

As you can see, this is very similar to just using the `redis-rs` crate directly. Instead of a `redis::Client`, the pool is passed in and we try to get a connection from the pool with a configured timeout.

In `set_str` and `get_str`, the only thing that changes is that `get_con` is called on each invocation of these functions. In the previous case, this would have meant creating a Redis connection for every single command, which would have been even more inefficient.

But since we have a pool with precreated connections, we can get them here and they can be shared between requests freely while no Redis action is going on in our current request.

Also, notice how the API is almost exactly the same, except for the `async` and `await` parts, so moving from one to the other is not too painful if that is something you have to do at some point.

## Using mobc (async)

Speaking of async, let’s look at the last of our three approaches: an async connection Pool.

As mentioned above, there are several crates for this. We’ll use [mobc](https://github.com/importcjj/mobc), mostly because I’ve used it successfully in a production setting already. Again, the other options work similarly and all seem like great options as well.

Let’s define our configurations and create the pool.

```rust
pub type MobcPool = Pool<RedisConnectionManager>;
pub type MobcCon = Connection<RedisConnectionManager>;

const CACHE_POOL_MAX_OPEN: u64 = 16;
const CACHE_POOL_MAX_IDLE: u64 = 8;
const CACHE_POOL_TIMEOUT_SECONDS: u64 = 1;
const CACHE_POOL_EXPIRE_SECONDS: u64 = 60;

pub async fn connect() -> Result<MobcPool> {
    let client = redis::Client::open(REDIS_CON_STRING).map_err(RedisClientError)?;
    let manager = RedisConnectionManager::new(client);
    Ok(Pool::builder()
        .get_timeout(Some(Duration::from_secs(CACHE_POOL_TIMEOUT_SECONDS)))
        .max_open(CACHE_POOL_MAX_OPEN)
        .max_idle(CACHE_POOL_MAX_IDLE)
        .max_lifetime(Some(Duration::from_secs(CACHE_POOL_EXPIRE_SECONDS)))
        .build(manager))
}
```

This is very similar to r2d2, which is not a coincidence; many connection pool crates took inspiration from r2d2’s excellent API.

To round the async pool implementation off, connections, getting, and setting:

```rust
async fn get_con(pool: &MobcPool) -> Result<MobcCon> {
    pool.get().await.map_err(|e| {
        eprintln!("error connecting to redis: {}", e);
        RedisPoolError(e).into()
    })
}

pub async fn set_str(pool: &MobcPool, key: &str, value: &str, ttl_seconds: usize) -> Result<()> {
    let mut con = get_con(&pool).await?;
    con.set(key, value).await.map_err(RedisCMDError)?;
    if ttl_seconds > 0 {
        con.expire(key, ttl_seconds).await.map_err(RedisCMDError)?;
    }
    Ok(())
}

pub async fn get_str(pool: &MobcPool, key: &str) -> Result<String> {
    let mut con = get_con(&pool).await?;
    let value = con.get(key).await.map_err(RedisCMDError)?;
    FromRedisValue::from_redis_value(&value).map_err(|e| RedisTypeError(e).into())
}
```

This should look quite familiar by now. The solution is a bit of a hybrid between the first and second options: The pool is passed in and fetches a connection at the start, but this time in an asynchronous way with `async` and `await`.

## Bringing it all together

Now that we’ve covered the basics and we have three isolated ways to connect to and interact with Redis, the next step is to bring all of them together in a warp web application.

Let’s start with the `main` function to get an overview of how the application is structured and what is needed to get this to run.

```rust
#[tokio::main]
async fn main() {
    let redis_client = redis::Client::open(REDIS_CON_STRING).expect("can create redis client");
    let mobc_pool = mobc_pool::connect().await.expect("can create mobc pool");
    let r2d2_pool = r2d2_pool::connect().expect("can create r2d2 pool");

    let direct_route = warp::path!("direct")
        .and(with_redis_client(redis_client.clone()))
        .and_then(direct_handler);

    let r2d2_route = warp::path!("r2d2")
        .and(with_r2d2_pool(r2d2_pool.clone()))
        .and_then(r2d2_handler);

    let mobc_route = warp::path!("mobc")
        .and(with_mobc_pool(mobc_pool.clone()))
        .and_then(mobc_handler);

    let routes = mobc_route.or(direct_route).or(r2d2_route);
    warp::serve(routes).run(([0, 0, 0, 0], 8080)).await;
}
```

It seems like not that much is required, that’s very little code!

At the start of the application, we’ll try to create a Redis client and the two connection pools. We’ll fail if any of them fail.

Next we’ll tackle the route definitions for each of the different approaches. These look quite similar to each other, with the only difference being the `with_xxx` filters.

```rust
fn with_redis_client(
    client: redis::Client,
) -> impl Filter<Extract = (redis::Client,), Error = Infallible> + Clone {
    warp::any().map(move || client.clone())
}

fn with_mobc_pool(
    pool: MobcPool,
) -> impl Filter<Extract = (MobcPool,), Error = Infallible> + Clone {
    warp::any().map(move || pool.clone())
}

fn with_r2d2_pool(
    pool: R2D2Pool,
) -> impl Filter<Extract = (R2D2Pool,), Error = Infallible> + Clone {
    warp::any().map(move || pool.clone())
}
```

The above filters are just simple extraction filters that make the passed-in things available to the handlers with which they are used. In each case, the client and pools are cloned and moved into the handlers - no magic here.

Last, but not least, let’s look at the actual handlers to see how the API we defined above can be used.

First, using `redis-rs` directly:

```rust
async fn direct_handler(client: redis::Client) -> WebResult<impl Reply> {
    let mut con = direct::get_con()
        .await
        .map_err(|e| warp::reject::custom(e))?;
    direct::set_str(&mut con, "hello", "direct_world", 60)
        .await
        .map_err(|e| warp::reject::custom(e))?;
    let value = direct::get_str(&mut con, "hello")
        .await
        .map_err(|e| warp::reject::custom(e))?;
    Ok(value)
}
```

As per the `with_direct` filter, we get the `redis::Client` passed in and immediately create a Redis connection with it.

Then, the handler writes `direct world` into the `hello` key inside of Redis and, immediately after, reads it from there.

The `r2d2` handler looks quite similar, but with an `R2D2Pool` getting passed in and without all the `async` and `await` stuff.

```rust
async fn r2d2_handler(pool: R2D2Pool) -> WebResult<impl Reply> {
    r2d2_pool::set_str(&pool, "r2d2_hello", "r2d2_world", 60)
        .map_err(|e| warp::reject::custom(e))?;
    let value = r2d2_pool::get_str(&pool, "r2d2_hello").map_err(|e| warp::reject::custom(e))?;
    Ok(value)
}
```

Of course, using a synchronous pool with an asynchronous web framework isn’t generally advisable. But there are plenty of frameworks that aren’t based on async/await, and for those, the synchronous pool will be perfect.

To finish it off, here’s the asynchronous pool using mobc:

```rust
async fn mobc_handler(pool: MobcPool) -> WebResult<impl Reply> {
    mobc_pool::set_str(&pool, "mobc_hello", "mobc_world", 60)
        .await
        .map_err(|e| warp::reject::custom(e))?;
    let value = mobc_pool::get_str(&pool, "mobc_hello")
        .await
        .map_err(|e| warp::reject::custom(e))?;
    Ok(value)
}
```

If you look at the three Redis client implementations — filters, routes, and handlers — you can see that they differ very little. That means you’ll be able to migrate from one to the other based on your needs, without having to rewrite everything from scratch.

It’s finally time to run the application.

```bash
cargo run
```

Next, start a local Redis instance with Docker.

```bash
docker run -p 6379:6379 redis:5.0
```

If you use `curl` to call the above-defined endpoints, you should see the correct responses.

```bash
curl http://localhost:8080/direct
curl http://localhost:8080/r2d2
curl http://localhost:8080/mobc
```

For very simple applications where you only rarely call into Redis, getting a new connection each time is perfectly fine. But if you have several Redis calls per requests with many requests happening at the same time, a connection pool will greatly increase performance.

When in doubt, test, profile, and make an informed decision based on the data you get.

You can find the full code for this example on [GitHub](https://github.com/zupzup/rust-redis-web-example).

## Conclusion

In this post, we explored three approaches to using Redis within a Rust web application. The most appropriate method for your application will depend on your use case and the way you structure the app. At any rate, the ability to switch between crates and approaches without having to change too much is a huge advantage.

The redis-rs crate, together with the rich ecosystem of both synchronous and asynchronous connection pools, is ready for production use and strikes a great balance between usability and performance.


