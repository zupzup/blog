*This post was originally posted on the [LogRocket](https://blog.logrocket.com/build-websocket-server-with-rust/) blog on 25.04.2023 and was cross-posted here by the author.*

In a previous post, we covered creating [an async CRUD web service in Rust using warp](https://blog.logrocket.com/create-async-crud-web-service-rust-with-warp/). However, another cool thing about warp is that it supports [WebSockets](https://blog.logrocket.com/websocket-tutorial-real-time-node-react/). This tutorial will demonstrate how to build a basic message relay service in Rust that clients can connect to via WebSockets.

After a client registers with their `user_id`, they get a unique connection ID they can use to connect via WebSockets. They are then able to receive real-time messages, which are published on the service. Clients can also communicate topics they are interested in via the WebSocket connection. For example, if they subscribe to the topics cats and dogs, they’ll only get published messages tagged with those topics. Published messages can be addressed directly to a specific `user_id` or broadcast to all users. Let’s get started, shall we?

## What is WebSocket?

WebSocket is an advanced API that allows two-way communication between the client’s web browser and server. This allows the server to communicate with clients in real time without forcing the client to poll the server for updates. Using Rust to build the WebSocket server enables the server to handle a large number of connections without compromising speed. This is because of Rust’s reliability and speed.

## Benefits of building a WebSocket server with Rust

A WebSocket server needs a large amount of memory to maintain active connections. Luckily, Rust’s [efficient memory management](https://blog.logrocket.com/using-cow-rust-efficient-memory-utilization/) makes it a good choice for WebSocket servers. Because most of the message passing is event-driven, Rust’s [async/await](https://blog.logrocket.com/asynchronous-i-o-and-async-await-packages-in-rust/) support makes the most efficient use of CPU time to increase application performance and throughput. Rust also provides a robust type system, a large set of production-grade WebSocket implementations, and predictable performance.

## Building a WebSocket server with Rust

All you need is a reasonably recent Rust installation (v1.39+) to follow along. You will also need a tool to test WebSockets connections, such as [websocat](https://github.com/vi/websocat), and a tool to [send HTTP requests](https://blog.logrocket.com/best-rust-http-client/), such as [curl](https://blog.logrocket.com/an-intro-to-curl-the-basics-of-the-transfer-tool/) or [Postman](https://blog.logrocket.com/how-automate-api-tests-postman/). First, create a new Rust project with the following command:

```bash
cargo new warp-ws-example
cd warp-ws-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need, as shown below:

```toml
[dependencies]
tokio = { version = "1.28", features = ["macros", "sync", "rt-multi-thread"] }
tokio-stream = "0.1.14"
warp = "0.3"
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0"
futures = { version = "0.3", default-features = false }
uuid = { version = "1.1.2", features = ["serde", "v4"] }
```

Nothing is shocking here. We need [warp](https://docs.rs/warp/latest/warp/) and [tokio](https://docs.rs/tokio/latest/tokio/) to run the web server and [serde_json](https://blog.logrocket.com/json-and-rust-why-serde_json-is-the-top-choice/) to serialize and deserialize JSON. The `uuid` crate will be used to create unique connection IDs, and `futures` crate will be helpful when dealing with the asynchronous data streams of the WebSocket.

## Understanding the data structures

Before we get started, let’s look at some of the data structures we’ll use to get some more context. First of all, the `Client` is at the core of this application. Here’s what that looks like:

```rust
pub struct Client {
    pub user_id: usize,
    pub topics: Vec<String>,
    pub sender: Option<mpsc::UnboundedSender<std::result::Result<Message, warp::Error>>>,
}
```

There is a difference between a client and a user in this case. A user can have several clients — think of the same user connecting to the API using a mobile app and a web app, for example. Clients have a `user_id`, a list of `topics` they’re interested in, and a `sender`. This sender sends part of an MPSC (multiple producers, single consumer) channel.

We’ll get to the sender later on, but suffice it to say for now that it’s used to send messages to this connected client via WebSockets. The following data structures are used in the [REST API](https://blog.logrocket.com/building-rest-api-rust-warp/) to register users and broadcast `events`:

```rust
#[derive(serde::Deserialize, serde::Serialize)]
pub struct RegisterRequest {
    user_id: usize,
}

#[derive(serde::Deserialize, serde::Serialize)]
pub struct RegisterResponse {
    url: String,
}

#[derive(serde::Deserialize, serde::Serialize)]
pub struct Event {
    topic: String,
    user_id: Option<usize>,
    message: String,
}
```

These are not particularly interesting. As mentioned above, `events` are tagged with a specific topic and can be addressed directly to a user, in which case the `user_id` will be set explicitly. Last but not least, we need a way for clients to communicate the topics they’re interested in. If they don’t set the topics explicitly, they’ll be defaulted to `cats` — because who doesn’t love those? Here’s what the code looks like:

```rust
#[derive(serde::Deserialize, serde::Serialize)]
pub struct TopicsRequest {
    topics: Vec<String>,
}
```

## Getting the server up and running

Now that you have a mental model of what we’ll build, let’s start by spinning up a warp web server with all the needed routes:

```rust
mod handler;
mod ws;

type Result<T> = std::result::Result<T, Rejection>;
type Clients = Arc<Mutex<HashMap<String, Client>>>;

#[tokio::main]
async fn main() {
  let clients: Clients = Arc::new(Mutex::new(HashMap::new()));

  let health_route = warp::path!("health").and_then(handler::health_handler);

  let register = warp::path("register");
  let register_routes = register
    .and(warp::post())
    .and(warp::body::json())
    .and(with_clients(clients.clone()))
    .and_then(handler::register_handler)
    .or(register
      .and(warp::delete())
      .and(warp::path::param())
      .and(with_clients(clients.clone()))
      .and_then(handler::unregister_handler));

  let publish = warp::path!("publish")
    .and(warp::body::json())
    .and(with_clients(clients.clone()))
    .and_then(handler::publish_handler);

  let ws_route = warp::path("ws")
    .and(warp::ws())
    .and(warp::path::param())
    .and(with_clients(clients.clone()))
    .and_then(handler::ws_handler);

  let routes = health_route
    .or(register_routes)
    .or(ws_route)
    .or(publish)
    .with(warp::cors().allow_any_origin());

  warp::serve(routes).run(([127, 0, 0, 1], 8000)).await;
}

fn with_clients(clients: Clients) -> impl Filter<Extract = (Clients,), Error = Infallible> + Clone {
  warp::any().map(move || clients.clone())
}
```

That’s quite a bit of code. Let’s go through it step by step. As mentioned above, we want clients to connect via WebSockets to our service. To accommodate this, we need a way to keep track of these clients within the service. We can solve this in many ways, but in this simple case, we’ll keep them around in memory — specifically, within a `HashMap`.

However, because this collection of clients needs to be accessed and mutated by several actors throughout the system (e.g., registering new clients, sending messages, updating topics, and more), we need to ensure it can be safely passed around between threads and [avoid data race](https://blog.logrocket.com/implementing-data-parallelism-rayon-rust/).

That’s why the `Clients` types are the first thing we defined above — an `Arc<Mutex<HashMap<String, Client>>>`. This type may look scary, but essentially, we want the map of connection IDs for clients behind a [Mutex](https://blog.logrocket.com/build-websocket-server-with-rust/%22https://blog.logrocket.com/understanding-handling-rust-mutex-poisoning/) so a single writer can only mutate it. To safely pass it to other threads, we wrap it into an `Arc`, an atomic smart pointer type that provides shared ownership by keeping a count of readers. So far, so good. In the main function, this `Clients` data structure is initialized, followed by a handful of route definitions:

* `GET /health`: Indicates if the service is up
* `POST /register`: Registers clients in the application
* `DELETE /register/{client_id}`: Unregisters the client with an ID
* `POST /publish`: Broadcasts an event to clients
* `GET /ws`: The WebSocket endpoint

The `with_clients` filter is created to pass the clients into these routes. This clones and passes a pointer to the clients into the routes using them. Besides, all handlers (except the WebSockets one) are pretty basic. For the `/ws` route, the `warp::ws()` filter is used, which makes it possible to upgrade the connection to a WebSocket connection in the handler. At the end of `main`, the routes are combined into a router with [CORS support](https://blog.logrocket.com/the-ultimate-guide-to-enabling-cross-origin-resource-sharing-cors/), and the server is started on `port 8000`.

## Registering clients

Now that the server is set up, let’s look at the handlers for the routes defined above, starting with client registration. To make the code a bit nicer to read, let’s put the handlers into a different file called `handler.rs`. Let’s start with registering a new client, where a JSON body with a `user_id` is sent to the service, like so:

```rust
pub async fn register_handler(body: RegisterRequest, clients: Clients) -> Result<impl Reply> {
  let user_id = body.user_id;
  let uuid = Uuid::new_v4().simple().to_string();

  register_client(uuid.clone(), user_id, clients).await;
  Ok(json(&RegisterResponse {
    url: format!("ws://127.0.0.1:8000/ws/{}", uuid),
  }))
}

async fn register_client(id: String, user_id: usize, clients: Clients) {
  clients.lock().await.insert(
    id,
    Client {
      user_id,
      topics: vec![String::from("cats")],
      sender: None,
    },
  );
}

pub fn health_handler() -> impl Future<Output = Result<impl Reply>> {
    futures::future::ready(Ok(StatusCode::OK))
}
```

The process of registering a new client is simple. First, a new uuid is created. This `ID` creates a new `Client` with an empty sender, the user’s `ID`, and default topics. These are simply added to the client’s data structure, returning a WebSocket URL with the uuid to the user. The user can connect the client via WebSockets with this URL.

To add the newly created client to the shared client structure, we need to `lock()` the `Mutex`. Since we’re using tokio’s asynchronous `Mutex` in this case, this is a `future` and should, therefore, be awaited. After the `lock` is acquired, simply `insert()` the new client into the underlying `HashMap`. Once the `lock` goes out of scope, it’s dropped, and others can access the data structure again. Great! We can call the `register` endpoint like this:

```bash
curl -X POST 'http://localhost:8000/register' -H 'Content-Type: application/json' -d '{ "user_id": 1 }'
{"url":"ws://127.0.0.1:8000/ws/625ac78b88e047a1bc7b3f8459702078"}
```

Unregistering clients is even easier. Check it out:

```rust
pub async fn unregister_handler(id: String, clients: Clients) -> Result<impl Reply> {
  clients.lock().await.remove(&id);
  Ok(StatusCode::OK)
}
```

The client with the given `ID` (the above-generated `uuid`) is simply removed from the `Clients` data structure. You might ask yourself, “What happens if you’re already connected via WebSockets using this ID?” They’re simply disconnected, and everything is closed and cleaned up on the side of the service.

This works because removing the `Client` makes it go out of scope, which means it gets dropped. This, in turn, drops the sending side of the channel within the client, which closes the channel and triggers an error. As we’ll see later on, this is a signal we can use to close the connection on our side. Calling `unregister` works like this:

```bash
curl -X DELETE 'http://localhost:8000/register/e2fa90682255472b9221709566dbceba'
```

## Connecting via WebSocket

Now that clients can register and unregister, it’s time to let them connect to our real-time WebSocket endpoint. Let’s start with the `handler`, as shown below:

```rust
pub async fn ws_handler(ws: warp::ws::Ws, id: String, clients: Clients) -> Result<impl Reply> {
  let client = clients.lock().await.get(&id).cloned();
  match client {
    Some(c) => Ok(ws.on_upgrade(move |socket| ws::client_connection(socket, id, clients, c))),
    None => Err(warp::reject::not_found()),
  }
}
```

First, the given `client ID` is checked against the `Clients` data structure. If no such client exists, a 404 error is returned. If a client is found, `ws.on_upgrade()` is used to upgrade the connection to a WebSocket connection, where the `ws::client_connection` function is called, like so:

```rust
pub async fn client_connection(ws: WebSocket, id: String, clients: Clients, mut client: Client) {
    let (client_ws_sender, mut client_ws_rcv) = ws.split();
    let (client_sender, client_rcv) = mpsc::unbounded_channel();

    let client_rcv = UnboundedReceiverStream::new(client_rcv);
    tokio::task::spawn(client_rcv.forward(client_ws_sender).map(|result| {
        if let Err(e) = result {
            eprintln!("error sending websocket msg: {}", e);
        }
    }));
```

This is the core part of the WebSockets logic, so let’s go through it slowly. The function gets a `warp::ws::WebSocket` passed into it by the `warp::ws filter`. You can loosely consider this the upgraded WebSocket connection and an asynchronous `Stream` and `Sink`. The `split()` function of `futures::StreamExt` splits this up into a `stream` and a `sink`, which can be considered a sender and a receiver.

Next, create an unbounded MPSC channel to send messages to the client. Also, if you remember the `sender` on the `Client` object, the `client_sender` is exactly this sender part of the channel. The next step is to spawn a tokio task in which the messages sent to the receiver part of the client (`client_rcv`) are propagated to the sender part of the WebSocket stream (`client_ws_sender`) using `futures::StreamExt::forward()`.

If sending the message fails, we log the error. In a different scenario, closing the connection at this point could also be feasible, depending on the error. The next step is to update the `client` with the newly created `sender`, like this:

```rust
client.sender = Some(client_sender);
clients.lock().await.insert(id.clone(), client);

println!("{} connected", id);
```

The passed-in `client` gets the `sender` part of the sending channel, and the data structure is updated. From now on, if someone sends anything to this sender end of the channel, it will be forwarded to the client via WebSocket, so we log the client as connected.

Now, we can send to a connected client, but we also want to be able to receive data on the WebSocket. First of all, the client should be able to send us pings to check if the connection is healthy, but we also want to enable clients to change their preferred topics via WebSocket. Here’s the code:

```rust
while let Some(result) = client_ws_rcv.next().await {
  let msg = match result {
    Ok(msg) => msg,
    Err(e) => {
      eprintln!("error receiving ws message for id: {}): {}", id.clone(), e);
      break;
      }
  };
  client_msg(&id, msg, &clients).await;
}

clients.lock().await.remove(&id);
  println!("{} disconnected", id);
}
```

To receive messages from the WebSocket, we can use the receiver end (`Stream`) of the WebSocket. Like any other async stream, we can simply wait for values using `.next().await` in a loop. When a message is received, that message is forwarded to the `client_msg` function, which we’ll look at next.

But, maybe more interestingly, if an error happens, that error is logged, and we break out of the loop, which leads to the end of the function. The only way to get here is if there is an error. In that case, we want to close the connection and remove the client from the shared data structure.

This is also where we’ll end up if a client is unregistered with a running connection. The only difference is that the client will already have been removed, triggering the error on the connection. To finish the WebSockets part, let’s look at the `client_msg` function, which deals with incoming messages from the client:

```rust
async fn client_msg(id: &str, msg: Message, clients: &Clients) {
  println!("received message from {}: {:?}", id, msg);
  let message = match msg.to_str() {
    Ok(v) => v,
    Err(_) => return,
  };

  if message == "ping" || message == "ping\n" {
    return;
  }

  let topics_req: TopicsRequest = match from_str(&message) {
    Ok(v) => v,
    Err(e) => {
      eprintln!("error while parsing message to topics request: {}", e);
      return;
    }
  };

  let mut locked = clients.lock().await;
  match locked.get_mut(id) {
    Some(v) => {
      v.topics = topics_req.topics;
    }
    None => return,
  };
}
```

First, the incoming message is logged, giving us a way to see what is coming in for testing. Next, the message is converted to a string. If it can’t be, we bail since we’re only interested in string messages. As mentioned earlier, clients should be able to send us pings, so if the message is `ping`, we simply return. We could also send back a `pong` or whatever we want to use in this case.

The other kind of messages we’re interested in are TopicsRequests, which are sent when a client wants to change their preferred topics. To do that, the client has to send some JSON to the WebSocket, which is then parsed to a list of topics. This list is then updated within the client’s data structure.

From then on, the client will only get messages according to their new topic preferences. Nice! WebSockets are now available for clients in the service. To test this, you can use a tool such as websocat like this:

```bash
websocat -t ws://127.0.0.1:8000/ws/2fd7f6e2e3294057b714c6c1c5aa827d
{ "topics": ["cats", "dogs"] }
```

## Relaying messages to clients

Almost done! The only piece of the puzzle we’re still missing is the ability to broadcast messages to connected clients. This is done using the `/publish` endpoint, as shown below:

```rust
pub async fn publish_handler(body: Event, clients: Clients) -> Result<impl Reply> {
  clients
    .lock()
    .await
    .iter_mut()
    .filter(|(_, client)| match body.user_id {
      Some(v) => client.user_id == v,
      None => true,
    })
    .filter(|(_, client)| client.topics.contains(&body.topic))
    .for_each(|(_, client)| {
      if let Some(sender) = &client.sender {
        let _ = sender.send(Ok(Message::text(body.message.clone())));
      }
    });

  Ok(StatusCode::OK)
}
```

When anyone wants to broadcast a message to clients, we have to iterate the client’s data structure if a `user_id` is set, filtering out all clients that are not the specified user. We’re only interested in clients that are subscribed to the topic of the message. We use each client’s `sender` to transmit the message down the pipeline. The publishing endpoint can be called like this:

```bash
curl -X POST 'http://localhost:8000/publish' \
-H 'Content-Type: application/json' \
-d '{"user_id": 1, "topic": "cats", "message": "are awesome"}'
```

This message will be sent to the connected clients with a `user_id` of `1`, subscribed to the topic `cats`. Perfect! Everything we set out to build is done and works nicely. You can find the complete code for this example on [GitHub](https://github.com/zupzup/warp-websockets-example).

## Converting `Mutex` to `RwLock`

Since there are multiple only-read operations on `Clients`, this data access pattern is optimized using a [RwLock](https://blog.logrocket.com/build-websocket-server-with-rust/%22https://blog.logrocket.com/an-introduction-to-profiling-a-rust-web-application/); that allows multiple readers simultaneously, i.e., multiple threads can read `Clients` in parallel. Using `RwLock` will reduce the application’s latency by reducing the time handlers wait for exclusive access to `Clients`, even if the handler only needs read access.

## Conclusion

WebSockets are fantastic, both for interactive, real-time web experiences and in combination with REST APIs to update the UI without the need for clients to poll for changes. Warp makes WebSockets easy to use, with the caveat that depending on the use case, some background knowledge of asynchronous streams and concurrency in Rust is required. But, since those are very useful skills within the area of Rust web development in general, that seems reasonable enough.

