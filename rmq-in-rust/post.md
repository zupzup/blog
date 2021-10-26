In this post we will take a look at how to integrate a Rust web application using [warp](https://github.com/seanmonstar/warp) with RabbitMQ. For this purpose, we will use the [lapin](https://github.com/CleverCloud/lapin) library together with [deadpool](https://github.com/bikeshedder/deadpool) for pooling connections.

The example we will build is pretty simple. There is an endpoint where you can send messages which are then sent to a RabbitMQ instance. At the same time, the service listens to new events coming in, logging them as they are taken from the queue.

## Example

Let's start with some types:

```rust
type WebResult<T> = StdResult<T, Rejection>;
type RMQResult<T> = StdResult<T, PoolError>;
type Result<T> = StdResult<T, Error>;

type Connection = deadpool::managed::Object<deadpool_lapin::Manager>;

#[derive(ThisError, Debug)]
enum Error {
    #[error("rmq error: {0}")]
    RMQError(#[from] lapin::Error),
    #[error("rmq pool error: {0}")]
    RMQPoolError(#[from] PoolError),
}

impl warp::reject::Reject for Error {}
```

These are just some helper types so we don't have to type as much, as well as a custom Error type, which implements `Reject`, so we can use it to return errors from `warp` endpoints.

Let's look at `main` next:

```rust
#[tokio::main]
async fn main() -> Result<()> {
    let addr = std::env::var("AMQP_ADDR")
        .unwrap_or_else(|_| "amqp://rmq:rmq@127.0.0.1:5672/%2f".into());
    let manager = Manager::new(addr, ConnectionProperties::default().with_tokio());
    let pool: Pool = deadpool::managed::Pool::builder(manager)
        .max_size(10)
        .build()
        .expect("can create pool");

    let health_route = warp::path!("health").and_then(health_handler);
    let add_msg_route = warp::path!("msg")
        .and(warp::post())
        .and(with_rmq(pool.clone()))
        .and_then(add_msg_handler);

    let routes = health_route
        .or(add_msg_route);

    println!("Started server at localhost:8000");
    let _ = join!(
        warp::serve(routes).run(([0, 0, 0, 0], 8000)),
        rmq_listen(pool.clone())
    );
    Ok(())
}
```

In this snippet we can already see the basic structure of the application. First, we create a RabbitMQ connection pool using `deadpool`. Then, we define some routes - one as a `/health` endpoint, and another for sending messages to the queue at `/msg`.

At the end of `main`, we start the `warp` web server and, at the same time, we use `futures::join!` to start the `rmq_listen` function. These two futures will run until one of them finishes, which, as we'll see, is only the case if there is an error.

Let's look at `rmq_listen` next:

```rust
async fn rmq_listen(pool: Pool) -> Result<()> {
    let mut retry_interval = tokio::time::interval(Duration::from_secs(5));
    loop {
        retry_interval.tick().await;
        println!("connecting rmq consumer...");
        match init_rmq_listen(pool.clone()).await {
            Ok(_) => println!("rmq listen returned"),
            Err(e) => eprintln!("rmq listen had an error: {}", e),
        };
    }
}
```

From a high level, we want `rmq_listen` to run as long as the web service is running. So if, for example, the RabbitMQ connection dies, or another error occurs during the connection's runtime, we want to reconnect and keep running. For this reason, the actual logic is wrapped with the above retry logic.

In `init_rmq_listen`, we actually listen to RabbitMQ events:

```rust
async fn init_rmq_listen(pool: Pool) -> Result<()> {
    let rmq_con = get_rmq_con(pool).await.map_err(|e| {
        eprintln!("could not get rmq con: {}", e);
        e
    })?;
    let channel = rmq_con.create_channel().await?;

    let queue = channel
        .queue_declare(
            "hello",
            QueueDeclareOptions::default(),
            FieldTable::default(),
        )
        .await?;
    println!("Declared queue {:?}", queue);

    let mut consumer = channel
        .basic_consume(
            "hello",
            "my_consumer",
            BasicConsumeOptions::default(),
            FieldTable::default(),
        )
        .await?;

    println!("rmq consumer connected, waiting for messages");
    while let Some(delivery) = consumer.next().await {
        if let Ok((channel, delivery)) = delivery {
            println!("received msg: {:?}", delivery);
            channel
                .basic_ack(delivery.delivery_tag, BasicAckOptions::default())
                .await?
        }
    }
    Ok(())
}
```

First, we get a RabbitMQ connection from the pool, create a channel and make sure the queue we want to listen to exists. Then we create a consumer for this queue.

This consumer is actually a `Stream`, so at the end of the function, we simply iterate over the stream. If any of the operations fail, or if there is nothing left to get from the stream, the outer `rmq_listen` function will restart the process. If we receive a message, we simply log it and acknowledge it.

OK, so now we have a way to receive messages from the queue. The only thing left is a way to send events to RabbitMQ using our `/msg` handler. In order to do that, we first need a way to pass the RabbitMQ pool to the handler:

```rust
fn with_rmq(pool: Pool) -> impl Filter<Extract = (Pool,), Error = Infallible> + Clone {
    warp::any().map(move || pool.clone())
}
```

The above filter simply clones the pool and makes this cloned reference available to any handler which has this filter. The actual `/msg` handler looks like this:

```rust
async fn add_msg_handler(pool: Pool) -> WebResult<impl Reply> {
    let payload = b"Hello world!";

    let rmq_con = get_rmq_con(pool).await.map_err(|e| {
        eprintln!("can't connect to rmq, {}", e);
        warp::reject::custom(Error::RMQPoolError(e))
    })?;

    let channel = rmq_con.create_channel().await.map_err(|e| {
        eprintln!("can't create channel, {}", e);
        warp::reject::custom(Error::RMQError(e))
    })?;

    channel
        .basic_publish(
            "",
            "hello",
            BasicPublishOptions::default(),
            payload.to_vec(),
            BasicProperties::default(),
        )
        .await
        .map_err(|e| {
            eprintln!("can't publish: {}", e);
            warp::reject::custom(Error::RMQError(e))
        })?
        .await
        .map_err(|e| {
            eprintln!("can't publish: {}", e);
            warp::reject::custom(Error::RMQError(e))
        })?;
    Ok("OK")
}
```

In this simple example, we send `Hello World` to everyone, but this could easily be adapted to send, for example, the payload of a `POST` request.

Again, we get a connection from the pool, then create a channel and publish a message on that channel, sending it to the `hello` queue.

One issue with the approach of connection-pooling in RabbitMQ is, that we still have to create a channel ever time and for optimal performance, both connections and channels would have to be pooled.

If you run this example now, together with a RabbitMQ server (e.g. with docker), you can call the `POST /msg` endpoint and observe the incoming messages from the queue in the listener.

To start a local RabbitMQ instance with the above credentials, you can use the following command:

```bash
docker run -p 15672:15672 -p 5672:5672 -e RABBITMQ_DEFAULT_USER=rmq -e RABBITMQ_DEFAULT_PASS=rmq  rabbitmq:3.8.4-management
```

The full example code can be found [here](https://github.com/zupzup/rmq-in-rust-example)

## Conclusion

Since the stabilization of async/await the ecosystem around distributed systems has grown and matured a lot and continues to do so.

With the 1.0 release of [lapin](https://github.com/CleverCloud/lapin), it seems like systems using RabbitMQ can now be built in a convenient and coherent way with Rust, which is a great step forward for the web ecosystem as well.

At this rate, a lacking ecosystem won't be an argument against using Rust in web services very soon! :)

#### Resources

* [Code Example](https://github.com/zupzup/rmq-in-rust-example)
* [lapin](https://github.com/CleverCloud/lapin)
* [deadpool](https://github.com/bikeshedder/deadpool)
* [warp](https://github.com/seanmonstar/warp)
