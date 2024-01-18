*This post was originally posted on the [LogRocket](https://blog.logrocket.com/libp2p-tutorial-build-a-peer-to-peer-app-in-rust/) blog on 14.12.2020 and was cross-posted here by the author.*

Over the last few years, due in large part to the hype surrounding blockchain and cryptocurrencies, decentralized applications have gained quite a bit of momentum. Another factor behind the surging interest in decentralization is greater awareness around the downsides of putting most of the web in the hands of a small cadre of companies in terms of data privacy and monopolization.

In any case, there have been some very interesting developments in the decentralized software scene recently, even aside from all the crypto and blockchain technology.

Notable examples include [IPFS](https://ipfs.io/); the brand new, distributed coding platform [Radicle](https://radicle.xyz/); the decentralized social network [Scuttlebutt](https://scuttlebutt.nz/); and many more applications within the [Fediverse](https://fediverse.party/), such as [Mastodon](https://joinmastodon.org/).

In this tutorial, we’ll show you how to build a very simple peer-to-peer application using Rust and the fantastic [libp2p](https://github.com/libp2p/rust-libp2p) library, which exists in different stages of maturity for a wide range of languages.

We’re going to build a cooking recipe app with a simple command-line interface that enables us to:

* Create recipes
* Publish recipes
* List local recipes
* List other peers we discovered on the network
* List published recipes of a given peer
* List all recipes of all peers we know

We’ll do this all in around 300 lines of Rust. Let’s get started!

## Installing Rust

To follow along, all you need is a recent Rust installation (1.47+).

First, create a new Rust project:

```bash
cargo new rust-p2p-example
cd rust-p2p-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need:

```toml
[dependencies]
libp2p = { version = "0.31", features = ["tcp-tokio", "mdns-tokio"] }
tokio = { version = "0.3", features = ["io-util", "io-std", "stream", "macros", "rt", "rt-multi-thread", "fs", "time", "sync"] }
serde = {version = "=1.0", features = ["derive"] }
serde_json = "1.0"
once_cell = "1.5"
log = "0.4"
pretty_env_logger = "0.4"
```

As mentioned above, we’ll use [libp2p](https://github.com/libp2p/rust-libp2p) for the peer-to-peer networking part. More specifically, we’re going to use it in concert with the Tokio async runtime. We’ll use [Serde for JSON serialization and deserialization](https://blog.logrocket.com/json-and-rust-why-serde_json-is-the-top-choice/) and a couple of helper libraries for logging and initializing state.

## What is `libp2p`?

[libp2p](https://libp2p.io/) is a set of protocols for building peer-to-peer applications that is focused on modularity.

There are library implementations for several languages, such as JavaScript, Go, and Rust. These libraries all implement the same `libp2p` specs, so a `libp2p` client built with Go can interact seamlessly with another client written in JavaScript, as long as they’re compatible in terms of the chosen protocol stack. These protocols cover a wide range, from basic network transport protocols to security layer protocols and multiplexing.

We won’t go too deep into the details of `libp2p` in this post, but if you’re interested to dive deeper, the [official libp2p docs](https://docs.libp2p.io/concepts/) offer a very nice overview of the various concepts we’ll encounter along the way.

## How `libp2p` works

To see `libp2p` in action, let’s start our recipe app. We’ll start by defining some constants and types we’ll need:

```rust
const STORAGE_FILE_PATH: &str = "./recipes.json";

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync + 'static>>;

static KEYS: Lazy<identity::Keypair> = Lazy::new(|| identity::Keypair::generate_ed25519());
static PEER_ID: Lazy<PeerId> = Lazy::new(|| PeerId::from(KEYS.public()));
static TOPIC: Lazy<Topic> = Lazy::new(|| Topic::new("recipes"));
```

We’ll store our local recipes in a simple JSON file called `recipes.json`, which the app will expect to be in the same folder as the executable. We also define a helper type for `Result`, which lets us propagate arbitrary errors.

Then, we use `once_cell::Lazy`, to lazily initialize a few things. First and foremost, we use it to generate a key pair and a so-called `PeerId` derived from the public key. We also create a `Topic`, which is another key concept from `libp2p`.

What does all this mean? In short, a `PeerId` is simply a unique identifier for a specific peer within the whole peer to peer network. We derive it from a key pair to ensure its uniqueness. Also, the key pair enables us to communicate securely with the rest of the network, making sure no one can impersonate us.

A `Topic`, on the other hand, is a concept from Floodsub, which is an implementation of `libp2p`’s [pub/sub interface](https://github.com/libp2p/specs/tree/master/pubsub). A `Topic` is something we can `subscribe` to and send messages on — for example, to only listen to a subset of the traffic on a pub/sub network.

We’ll also need some types for the recipe:

```rust
type Recipes = Vec<Recipe>;

#[derive(Debug, Serialize, Deserialize)]
struct Recipe {
    id: usize,
    name: String,
    ingredients: String,
    instructions: String,
    public: bool,
}
```

And some types for the messages we plan to send around:

```rust
#[derive(Debug, Serialize, Deserialize)]
enum ListMode {
    ALL,
    One(String),
}

#[derive(Debug, Serialize, Deserialize)]
struct ListRequest {
    mode: ListMode,
}

#[derive(Debug, Serialize, Deserialize)]
struct ListResponse {
    mode: ListMode,
    data: Recipes,
    receiver: String,
}

enum EventType {
    Response(ListResponse),
    Input(String),
}
```

The recipe is rather straightforward. It has an ID, a name, some ingredients, and instructions to execute it. Also, we add a `public` flag so we can distinguish which recipes we want to share and which ones we want to keep for ourselves.

As mentioned in the beginning, there are two ways we can fetch lists from other peers: from all or from one, which is represented by the `ListMode` enum.

The `ListRequest` and `ListResponse` types are just wrappers for this type and the date sent using them.

The `EventType` enum distinguishes between a response from another peer and input from ourselves. We’ll see later on why this difference is relevant.

## Creating a `libp2p` client

Let’s start writing the `main` function to set up a peer within a peer-to-peer network.

```rust
#[tokio::main]
async fn main() {
    pretty_env_logger::init();

    info!("Peer Id: {}", PEER_ID.clone());
    let (response_sender, mut response_rcv) = mpsc::unbounded_channel();

    let auth_keys = Keypair::<X25519Spec>::new()
        .into_authentic(&KEYS)
        .expect("can create auth keys");
```

We initialize logging and create an async `channel` to communicate between different parts of the application. We’ll use this channel later on to send responses from the `libp2p` network stack back to our application to handle them.

Also, we create some authentication keys for the [Noise](https://noiseprotocol.org/) crypto protocol, which we’ll use to secure the traffic within the network. For this purpose, we create a new key pair and sign it with our identity keys using the `into_authentic` function.

The next step is important and involves some core concepts of `libp2p`: creating a so-called [Transport](https://docs.rs/libp2p/latest/libp2p/trait.Transport.html).

```rust
    let transp = TokioTcpConfig::new()
        .upgrade(upgrade::Version::V1)
        .authenticate(NoiseConfig::xx(auth_keys).into_authenticated())
        .multiplex(mplex::MplexConfig::new())
        .boxed();
```

A transport is a set of network protocols that enables connection-oriented communication between peers. It’s also possible to use multiple transports within one application — for example, TCP/IP and and Websockets, or UDP at the same time for different use cases.

In this example, we’ll use TCP as a base using Tokio’s async TCP. Once a TCP connection has been established, we’ll `upgrade` it to use `Noise` for secure communication. A web-based example of this would be to use TLS on top of HTTP to create a secure connection.

We use the `NoiseConfig:xx` handshake pattern, which is one of three options, because it’s the only one that is guaranteed to be interoperable with other `libp2p` applications.

The nice thing about `libp2p` is that we could write a Rust client and another one could write a JavaScript client, and they could still easily communicate as long as the protocols are implemented in both versions of the library.

At the end, we also [multiplex](https://docs.libp2p.io/concepts/stream-multiplexing/) the transport, which enables us to multiplex multiple substreams, or connections, on the same transport.

Phew, that’s quite a bit of theory! But all of this is can be found in the [libp2p docs](https://docs.libp2p.io/). This is just one of many, many ways to create a peer-to-peer transport.

The next concept is a `NetworkBehaviour`. This is the part within `libp2p` that actually defines the logic of the network and all peers - for example, what to do with incoming events and which events to send.

```rust
    let mut behaviour = RecipeBehaviour {
        floodsub: Floodsub::new(PEER_ID.clone()),
        mdns: TokioMdns::new().expect("can create mdns"),
        response_sender,
    };

    behaviour.floodsub.subscribe(TOPIC.clone());
```

In this case, as mentioned above, we’ll use the `FloodSub` protocol to deal with events. We’ll also use [mDNS](https://tools.ietf.org/html/rfc6762), which is a protocol for discovering other peers on the local network. We’ll also put the sender part of our channel in here so we can use it to propagate events back to the main part of the application.

The `FloodSub` topic we created earlier is now being subscribed to from our behavior, which means we’ll receive events and can send events on that topic.

We’re almost done with the `libp2p` setup. The last concept we need is the `Swarm`.

```rust
    let mut swarm = SwarmBuilder::new(transp, behaviour, PEER_ID.clone())
        .executor(Box::new(|fut| {
            tokio::spawn(fut);
        }))
        .build();
```

A [Swarm](https://docs.rs/libp2p/latest/libp2p/index.html#swarm) manages the connections created using the transport and executes the network behavior we created, triggering and receiving events and giving us a way to get to them from the outside.

We create the `Swarm` with our transport, behavior, and peer ID. The `executor` part simply tells the `Swarm` to use the `Tokio` runtime to run internally, but we could also use other async runtimes here.

The only thing left to do is to start our `Swarm`:

```rust
    Swarm::listen_on(
        &mut swarm,
        "/ip4/0.0.0.0/tcp/0"
            .parse()
            .expect("can get a local socket"),
    )
    .expect("swarm can be started");
```

Similar to starting, for example, a TCP server, we simply call `listen_on` with a local IP, letting the OS decide the port for us. This will start the `Swarm` with all of our setup, but we haven’t actually defined any logic yet.

Let’s start with handling user input.

## Handling input in `libp2p`

For user input, we’ll simply rely on good old STDIN. So before the `Swarm::listen_on` call, we’ll add:

```rust
    let mut stdin = tokio::io::BufReader::new(tokio::io::stdin()).lines();
```

This defined an async reader on STDIN, which reads the stream line by line. So if we press enter, there will be a new incoming message.

The next part is to create our event loop, which will listen to events from STDIN, from the Swarm, and from our response channel defined above.

```rust
    loop {
        let evt = {
            tokio::select! {
                line = stdin.next_line() => Some(EventType::Input(line.expect("can get line").expect("can read line from stdin"))),
                event = swarm.next() => {
                    info!("Unhandled Swarm Event: {:?}", event);
                    None
                },
                response = response_rcv.recv() => Some(EventType::Response(response.expect("response exists"))),
            }
        };
        ...
    }
}
```

We use Tokio’s `select` macro to wait for several async processes, handling the first one that finishes. We don’t do anything with the `Swarm` events; these are handled within our `RecipeBehaviour`, which we’ll see later on, but we still need to call `swarm.next()` to drive the `Swarm` forward.

Let’s add some event handling logic in place of the …:

```rust
        if let Some(event) = evt {
            match event {
                EventType::Response(resp) => {
                   ...
                }
                EventType::Input(line) => match line.as_str() {
                    "ls p" => handle_list_peers(&mut swarm).await,
                    cmd if cmd.starts_with("ls r") => handle_list_recipes(cmd, &mut swarm).await,
                    cmd if cmd.starts_with("create r") => handle_create_recipe(cmd).await,
                    cmd if cmd.starts_with("publish r") => handle_publish_recipe(cmd).await,
                    _ => error!("unknown command"),
                },
            }
        }
```

If there is an event, we match on it and see if it’s a `Response` or an `Input` event. Let’s look at the `Input` events only for now.

There are a couple of options. We support the following commands:

* `ls p` lists all known peers
* `ls r` lists local recipes
* `ls r {peerId}` lists published recipes from a certain peer
* `ls r all` lists published recipes from all known peers
* `publish r {recipeId}` publishes a given recipe
* `create r {recipeName}|{recipeIngredients}|{recipeInstructions}` creates a new recipe with the given data and an incrementing ID

Listing all recipes from peers, in this case, means sending a request for the recipes to our peers, waiting for them to respond, and displaying the results. In a peer-to-peer network, this can take a while since some peers might be on the other side of the planet and we don’t know if all of them will even respond to us. This is quite different from sending a request to an HTTP server, for example.

Let’s look at the logic for listing peers first:

```rust
async fn handle_list_peers(swarm: &mut Swarm<RecipeBehaviour>) {
    info!("Discovered Peers:");
    let nodes = swarm.mdns.discovered_nodes();
    let mut unique_peers = HashSet::new();
    for peer in nodes {
        unique_peers.insert(peer);
    }
    unique_peers.iter().for_each(|p| info!("{}", p));
}
```

In this case, we can use `mDNS` to give us all discovered nodes, iterating and displaying them. Easy.

Let’s walk through creating and publishing recipes next, before tackling the list commands:

```rust
async fn handle_create_recipe(cmd: &str) {
    if let Some(rest) = cmd.strip_prefix("create r") {
        let elements: Vec<&str> = rest.split("|").collect();
        if elements.len() < 3 {
            info!("too few arguments - Format: name|ingredients|instructions");
        } else {
            let name = elements.get(0).expect("name is there");
            let ingredients = elements.get(1).expect("ingredients is there");
            let instructions = elements.get(2).expect("instructions is there");
            if let Err(e) = create_new_recipe(name, ingredients, instructions).await {
                error!("error creating recipe: {}", e);
            };
        }
    }
}

async fn handle_publish_recipe(cmd: &str) {
    if let Some(rest) = cmd.strip_prefix("publish r") {
        match rest.trim().parse::<usize>() {
            Ok(id) => {
                if let Err(e) = publish_recipe(id).await {
                    info!("error publishing recipe with id {}, {}", id, e)
                } else {
                    info!("Published Recipe with id: {}", id);
                }
            }
            Err(e) => error!("invalid id: {}, {}", rest.trim(), e),
        };
    }
}
```

In both cases, we need to parse the string to get to the `|`-separated data, or the given recipe ID in the case of `publish`, logging an error if the given input isn’t valid.

In the `create` case, we call the `create_new_recipe` helper function with the given data. Let’s check out all the helper functions we’ll need to interact with our simple local JSON storage for recipes:

```rust
async fn create_new_recipe(name: &str, ingredients: &str, instructions: &str) -> Result<()> {
    let mut local_recipes = read_local_recipes().await?;
    let new_id = match local_recipes.iter().max_by_key(|r| r.id) {
        Some(v) => v.id + 1,
        None => 0,
    };
    local_recipes.push(Recipe {
        id: new_id,
        name: name.to_owned(),
        ingredients: ingredients.to_owned(),
        instructions: instructions.to_owned(),
        public: false,
    });
    write_local_recipes(&local_recipes).await?;

    info!("Created recipe:");
    info!("Name: {}", name);
    info!("Ingredients: {}", ingredients);
    info!("Instructions:: {}", instructions);

    Ok(())
}

async fn publish_recipe(id: usize) -> Result<()> {
    let mut local_recipes = read_local_recipes().await?;
    local_recipes
        .iter_mut()
        .filter(|r| r.id == id)
        .for_each(|r| r.public = true);
    write_local_recipes(&local_recipes).await?;
    Ok(())
}

async fn read_local_recipes() -> Result<Recipes> {
    let content = fs::read(STORAGE_FILE_PATH).await?;
    let result = serde_json::from_slice(&content)?;
    Ok(result)
}

async fn write_local_recipes(recipes: &Recipes) -> Result<()> {
    let json = serde_json::to_string(&recipes)?;
    fs::write(STORAGE_FILE_PATH, &json).await?;
    Ok(())
}
```

The most basic building blocks are `read_local_recipes` and `write_local_recipes`, which simply read and deserialize or serialize and write recipes from or to the storage file.

The `publish_recipe` helper fetches all recipes from the file, looks for the recipe with the given ID, and sets its `public` flag to true.

When creating a recipe, we also fetch all recipes from the file, add a new recipe at the end, and write back the whole data, overriding the file. This is not super-efficient, but it’s simple and it works.

## Sending messages with libp2p

Let’s look at the `list` commands next and explore how we can send messages to other peers.

In the `list` command, there are three possible cases:

```rust
async fn handle_list_recipes(cmd: &str, swarm: &mut Swarm<RecipeBehaviour>) {
    let rest = cmd.strip_prefix("ls r ");
    match rest {
        Some("all") => {
            let req = ListRequest {
                mode: ListMode::ALL,
            };
            let json = serde_json::to_string(&req).expect("can jsonify request");
            swarm.floodsub.publish(TOPIC.clone(), json.as_bytes());
        }
        Some(recipes_peer_id) => {
            let req = ListRequest {
                mode: ListMode::One(recipes_peer_id.to_owned()),
            };
            let json = serde_json::to_string(&req).expect("can jsonify request");
            swarm.floodsub.publish(TOPIC.clone(), json.as_bytes());
        }
        None => {
            match read_local_recipes().await {
                Ok(v) => {
                    info!("Local Recipes ({})", v.len());
                    v.iter().for_each(|r| info!("{:?}", r));
                }
                Err(e) => error!("error fetching local recipes: {}", e),
            };
        }
    };
}
```

We parse the incoming command, stripping the `ls r` part, and checking what’s left. If there is nothing else in the command, we can simply fetch our local recipes and print them using the helpers defined in the previous section.

If we encounter the `all` keyword, we create a `ListRequest` with the `ListMode::ALL` set, serialize it to JSON, and, using the `FloodSub` instance within our `Swarm`, publish it to the previously mentioned `Topic`.

The same thing happens if we encounter a peer ID in the command, in which case we’d just send the `ListMode::One` mode with that peer ID. We could check whether it’s a valid peer ID, or whether it’s even a peer ID we have discovered, but let’s keep it simple: if there is no one to listen to it, nothing happens.

That’s all we need to do to send messages to the network. Now the question is, what happens with those messages? Where are they handled?

In the case of a peer-to-peer application, remember that we’re both the `Sender` and `Receiver` of events, so we need to deal with both outgoing and incoming events in our implementation.

## Responding to messages with `libp2p`

This is finally the part where our `RecipeBehaviour` comes in. Let’s define it:

```rust
#[derive(NetworkBehaviour)]
struct RecipeBehaviour {
    floodsub: Floodsub,
    mdns: TokioMdns,
    #[behaviour(ignore)]
    response_sender: mpsc::UnboundedSender<ListResponse>,
}
```

The behavior itself is simply a struct, but we use `libp2p`’s `NetworkBehaviour` derive macro, so we don’t have to manually implement all the trait functions ourselves.

This derive macro implements the [NetworkBehaviour](https://docs.rs/libp2p/latest/libp2p/swarm/trait.NetworkBehaviour.html) trait’s functions for all the members of the struct, which are not annotated with `behaviour(ignore)`. Our channel is ignored here because it has nothing to do with our behavior directly.

What’s left is to implement the `inject_event` function for both `FloodsubEvent` and `MdnsEvent`.

Let’s start with `mDNS`:

```rust
impl NetworkBehaviourEventProcess<MdnsEvent> for RecipeBehaviour {
    fn inject_event(&mut self, event: MdnsEvent) {
        match event {
            MdnsEvent::Discovered(discovered_list) => {
                for (peer, _addr) in discovered_list {
                    self.floodsub.add_node_to_partial_view(peer);
                }
            }
            MdnsEvent::Expired(expired_list) => {
                for (peer, _addr) in expired_list {
                    if !self.mdns.has_node(&peer) {
                        self.floodsub.remove_node_from_partial_view(&peer);
                    }
                }
            }
        }
    }
}
```

The `inject_event` function is called when an event for this handler comes in. On the `mDNS`  side, there are only two events, `Discovered` and `Expired`, which are triggered when we see a new peer on the network or when an existing peer goes away. In both cases, we either add it to or remove it from our `FloodSub` “partial view,” which is a list of nodes to propagate our messages to.

The `inject_event` for pub/sub events is a bit more complex. We need to react on both incoming `ListRequest` and `ListResponse` payloads. If we send a `ListRequest`, the peer that receives the request will fetch its local, published recipes and then will need a way to send them back.

The only way to send them back to the requesting peer is to publish them on the network. Since pub/sub is indeed the only mechanism we have, we need to react both to incoming requests and incoming responses.

Let’s see how this works:

```rust
    impl NetworkBehaviourEventProcess<FloodsubEvent> for RecipeBehaviour {
        fn inject_event(&mut self, event: FloodsubEvent) {
            match event {
                FloodsubEvent::Message(msg) => {
                    if let Ok(resp) = serde_json::from_slice::<ListResponse>(&msg.data) {
                        if resp.receiver == PEER_ID.to_string() {
                            info!("Response from {}:", msg.source);
                            resp.data.iter().for_each(|r| info!("{:?}", r));
                        }
                    } else if let Ok(req) = serde_json::from_slice::<ListRequest>(&msg.data) {
                        match req.mode {
                            ListMode::ALL => {
                                info!("Received ALL req: {:?} from {:?}", req, msg.source);
                                respond_with_public_recipes(
                                    self.response_sender.clone(),
                                    msg.source.to_string(),
                                );
                            }
                            ListMode::One(ref peer_id) => {
                                if peer_id == &PEER_ID.to_string() {
                                    info!("Received req: {:?} from {:?}", req, msg.source);
                                    respond_with_public_recipes(
                                        self.response_sender.clone(),
                                        msg.source.to_string(),
                                    );
                                }
                            }
                        }
                    }
                }
                _ => (),
            }
        }
    }
```

We match on the incoming message, trying to deserialize it to a request or response. In the case of a response, we simply print the response with the caller’s peer ID, which we get using `msg.source`. When we receive an incoming request, we need to differentiate between the `ALL` and `One` cases.

In the `One` case, we check if the given peer ID is the same as ours – that the request is actually meant for us. If it is, we return our published recipes, which is our response in the case of `ALL` as well.

In both cases, we call the `respond_with_public_recipes` helper:

```rust
fn respond_with_public_recipes(sender: mpsc::UnboundedSender<ListResponse>, receiver: String) {
    tokio::spawn(async move {
        match read_local_recipes().await {
            Ok(recipes) => {
                let resp = ListResponse {
                    mode: ListMode::ALL,
                    receiver,
                    data: recipes.into_iter().filter(|r| r.public).collect(),
                };
                if let Err(e) = sender.send(resp) {
                    error!("error sending response via channel, {}", e);
                }
            }
            Err(e) => error!("error fetching local recipes to answer ALL request, {}", e),
        }
    });
}
```

In this helper method, we use Tokio’s spawn to asynchronously execute a future, which reads all local recipes, creates a `ListResponse` from the data, and sends this data through the `channel_sender` over to our event loop, where we handle it like this:

```rust
                    EventType::Response(resp) => {
                        let json = serde_json::to_string(&resp).expect("can jsonify response");
                        swarm.floodsub.publish(TOPIC.clone(), json.as_bytes());
                    }
```

If we notice an “internally” sent over `Response` event, we serialize it to JSON and send it onto the network.

## Let’s see if it works

That’s it for the implementation. Now let’s test it.

To check that our implementation works, let’s start the application in several terminals using this command:

```bash
    RUST_LOG=info cargo run
```

Keep in mind that the application expects a file called `recipes.json` in the directory you’re starting it from.

When the application started, we get the following log, printing our peer ID:

```bash
    INFO  rust_peer_to_peer_example > Peer Id: 12D3KooWDc1FDabQzpntvZRWeDZUL351gJRy3F4E8VN5Gx2pBCU2
```

Now we need to press enter to start the event loop.

Upon entering `ls p`, we get a list of our discovered peers:

```bash
    ls p
     INFO  rust_peer_to_peer_example > Discovered Peers:
     INFO  rust_peer_to_peer_example > 12D3KooWCK6X7mFk9HeWw69WF1ueWa3XmphZ2Mu7ZHvEECj5rrhG
     INFO  rust_peer_to_peer_example > 12D3KooWLGN85pv5XTDALGX5M6tRgQtUGMWXWasWQD6oJjMcEENA
```

With `ls r`, we get the local recipes:

```bash
    ls r
     INFO  rust_peer_to_peer_example > Local Recipes (3)
     INFO  rust_peer_to_peer_example > Recipe { id: 0, name: " Coffee", ingredients: "Coffee", instructions: "Make Coffee", public: true }
     INFO  rust_peer_to_peer_example > Recipe { id: 1, name: " Tea", ingredients: "Tea, Water", instructions: "Boil Water, add tea", public: false }
     INFO  rust_peer_to_peer_example > Recipe { id: 2, name: " Carrot Cake", ingredients: "Carrots, Cake", instructions: "Make Carrot Cake", public: true }
```

Calling `ls r all` triggers sending a request to the other peers and returns their recipes:

```bash
    ls r all
     INFO  rust_peer_to_peer_example > Response from 12D3KooWCK6X7mFk9HeWw69WF1ueWa3XmphZ2Mu7ZHvEECj5rrhG:
     INFO  rust_peer_to_peer_example > Recipe { id: 0, name: " Coffee", ingredients: "Coffee", instructions: "Make Coffee", public: true }
     INFO  rust_peer_to_peer_example > Recipe { id: 2, name: " Carrot Cake", ingredients: "Carrots, Cake", instructions: "Make Carrot Cake", public: true }
```

The same happens if we use `ls r` with a peer ID:

```bash
    ls r 12D3KooWCK6X7mFk9HeWw69WF1ueWa3XmphZ2Mu7ZHvEECj5rrhG
     INFO  rust_peer_to_peer_example > Response from 12D3KooWCK6X7mFk9HeWw69WF1ueWa3XmphZ2Mu7ZHvEECj5rrhG:
     INFO  rust_peer_to_peer_example > Recipe { id: 0, name: " Coffee", ingredients: "Coffee", instructions: "Make Coffee", public: true }
     INFO  rust_peer_to_peer_example > Recipe { id: 2, name: " Carrot Cake", ingredients: "Carrots, Cake", instructions: "Make Carrot Cake", public: true }
```

It works! You can also try this with a huge amount of clients in the same network.

You can find the full example code at [GitHub](https://github.com/zupzup/rust-peer-to-peer-example).

## Conclusion

In this post we covered how to build a small, decentralized network application using Rust and `libp2p`.

If you’re coming from a web background, many of the networking concepts will be somewhat familiar, but building a peer-to-peer application still demands a fundamentally different approach to design and build.

The `libp2p` library is quite mature and, due to Rust’s popularity within the crypto scene, there is an emerging, rich ecosystem of libraries to build powerful decentralized applications.






