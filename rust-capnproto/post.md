In this post, we're going to look at how to use the wonderful [Cap'n Proto](https://capnproto.org/) serialization system in Rust.

I've been wanting to check out Cap'n Proto for a while now, since it has some very cool features and posts some impressive performance numbers. The little example shown in this article is a nice way to scratch the surface.

For this purpose, we're going to build a short example application, which will:

1. Define a `Cat` schema as our data model with a few fields, some nested objects, as well as some binary data in the form of an image
2. Implement serialization and deserialization of said data model using Cap'n Proto as well as JSON
3. Implement a simple TCP client which serializes some dummy data and sends it across the network
4. Implement a simple TCP Server for JSON and Cap'n Proto, which receive this data and deserialize it

We'll also add some primitive timing in the code, to see roughly how serialization and deserialization between Cap'n Proto and JSON compare and we'll log the size of the created payloads.

The point here is not to do a comparison between JSON and Cap'n Proto, because that would be comparing apples to oranges and we'd have to do some in-depth benchmarks. What I'm interested here is rather how the ergonomics compare and to get an order-of-magnitude baseline regarding payload size and performance for this basic example.

Let's start!

## Setup

Install the `capnp` tool for your platform as described [here](https://capnproto.org/install.html). You can check out [this](https://capnproto.org/capnp-tool.html) piece of documentation if you're interested in what you can do with it and how to use it.

For our purposes, it's enough to have it installed, since it'll be used during the build process of our little toy application.

Then, we create a new project using `cargo init` and edit `Cargo.toml` to include the following dependencies:

* `capnp`
* `capnpc`
* `tokio`
* `serde`
* `serde_json`
* `base64`

We're using Cap'n Proto, so we need both the runtime and build library for it. Also, since we're building a networked app, we'll use Tokio to build a simple server and client implementation and we'll need Serde to implement our JSON baseline for comparison. And finally, we'll use the `base64` crate to encode the binary image data, so we can send it as a String within the JSON payload.

With that out of the way, let's start by building our Cap'n Proto schema and include it in our build process.

## Schema and build.rs

First, we use the following command

```bash
capnp id
```

which in my case returned `@0x9d62d5f53cf92b66`. This is our unique ID for the `Cat` type.

Then, we create the schema file in `schemas/cats.capnp`.

```bash
@0x9d62d5f53cf92b66;

struct Cat {
    name @0: Text;
    age @1: UInt8;
    color @2: Text;
    cuteness @3: Float32;
    addresses@4: List(Address);

    struct Address {
        street @0: Text;
        number @1: UInt8;
        postalcode @2: UInt16;
    }

    image @5: Data;
}
```

The schema starts with the unique type id and then defines the data model in Cap'n Proto [schema language](https://capnproto.org/language.html). We define the names of the fields with the order in which they were added to the protocol (to maintain backwards-compatibility), as well as their data type.

We can also define nested structures directly inside the containing structures, being able to reference them within e.g. a `List` as can be seen at the `addresses` example. This is a very simple example and the schema language is quite powerful, so if you're interested in that, it would certainly make sense to dive a bit deeper into the docs and to check out some examples.
There is also a section on how to evolve the protocol avoiding breaking changes.

With the schema defined, it's time to add the Cap'n Proto compiler to our build process, so our schema is checked and built with every build of the project.

```rust
extern crate capnpc;

fn main() {
    capnpc::CompilerCommand::new()
        .output_path("src/")
        .src_prefix("schemas")
        .file("schemas/cats.capnp")
        .run()
        .expect("capnp compiles");
}
```

We create a `build.rs` file, which will be run during `cargo build`, which invokes the Cap'n Proto compiler with the parameters given above for our `Cat` schema.

Now we can use `cargo build` to both build our project and in addition, trigger the Cap'n Proto code generation for our defined schema. The generated file, in this case, will be created at `src/cats_capnp.rs` and it can be imported using `mod cats_capnp` into `main.rs`.

Next, let's implement the Cap'n Proto serialization and de-serialization logic.

## Cap'n Proto Serialization and Deserialization

Let's start by implementing serialization and deserialization using Cap'n Proto, starting with the logic to build a message.

```rust
fn build_cpnproto_msg(img: &[u8]) -> Vec<u8> {
    let mut msg = Builder::new_default();

    let mut cat = msg.init_root::<cats_capnp::cat::Builder>();
    cat.set_name("Minka");
    cat.set_age(8);
    cat.set_color("lucky");
    cat.set_cuteness(100.0);
    cat.set_image(img);
    let mut addresses = cat.init_addresses(10);
    {
        for i in 0..10 {
            let mut address = addresses.reborrow().get(i);
            address.set_street("some street");
            address.set_number(i as u8);
            address.set_postalcode(1234);
        }
    }

    let start = Instant::now();
    let data = serialize::write_message_to_words(&msg);
    let duration = start.elapsed();
    println!("CAPNP SE: duration: {:?}, len: {}", duration, data.len());
    data
}
```

The first step is to create a `Builder` and to initialize it with our `cats_capnp` builder, so we can start creating an instance of a cat. Then, we set each field using the generated setters. Then, to handle the nested data structure in addresses, we first use `init_addresses` with the amount of addresses we want to add and then open a new scope using `{}`.
This technique is necessary, so we can work with the addresses e.g. in a loop, by re-borrowing it, without running into scoping and borrow-checker issues.

Within this scope, we initialize the 10 addresses using a loop. Afterwards, we use `serialize::write_message_to_words` to create the data payload, measuring how long this takes and printing the duration, as well as the payload size. We don't have to do anything to the binary data of the image, since Cap'n Proto is a binary format.

Next, let's implement deserialization.

```rust
fn deserialize_cpnproto(data: &[u8]) {
    let start = Instant::now();

    let reader = serialize::read_message(data, ReaderOptions::new()).expect("can create reader");
    let cat = reader
        .get_root::<cats_capnp::cat::Reader>()
        .expect("can deserialize cat");

    let duration = start.elapsed();
    println!(
        "CAPNP DE: {} duration: {:?}",
        cat.get_name().unwrap(),
        duration
    );
}
```

To deserialize, we create a `Reader` from the incoming data using `serialize::read_message`, which we can then use to deserialize back a `Cat` type using `cats_capnp::cat::Reader`. Again, we measure the duration of this operation.

## JSON Serialization and Deserialization

Next, we'll implement the same serialization and deserialization using JSON.

First, we define the same types as in Cap'n Proto, but using structs and Serde.

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Address {
    street: String,
    number: u16,
    postalcode: u16,
}

#[derive(Serialize, Deserialize, Debug)]
struct Cat {
    name: String,
    age: u8,
    color: String,
    cuteness: f32,
    addresses: Vec<Address>,
    image: Option<String>,
}
```

The data model simply matches the schema we defined using Cap'n Proto, deriving the `Serialize` and `Deserialize` traits, so we can use our structs with Serde.

Now we can implement the JSON deserialization logic.

```rust
fn build_msg_json(img: &[u8]) -> Vec<u8> {
    let mut addresses = vec![];
    for i in 0..10 {
        addresses.push(Address {
            street: String::from("some street"),
            number: i,
            postalcode: 1234,
        });
    }

    let encoded_img = general_purpose::STANDARD_NO_PAD.encode(img);

    let cat = Cat {
        name: String::from("Minka"),
        age: 8,
        color: String::from("lucky"),
        cuteness: 100.0,
        addresses,
        image: Some(encoded_img),
    };

    let start = Instant::now();
    let data = serde_json::to_vec(&cat).expect("can json serialize cat");
    let duration = start.elapsed();
    println!("JSON SE: duration: {:?}, len: {}", duration, data.len());
    data
}
```

For the JSON serialization, we build our data model, again with 10 addresses nested in it and we encode the image data as base64, so we can send it as a String. Then, we simply serialize it using `serde_json` and print out the time this took, as well as the size of the payload.


Next, we'll implement JSON deserialization.

```rust
fn deserialize_json(data: &[u8]) {
    let start = Instant::now();
    let cat: Cat = serde_json::from_slice(data).expect("can deserialize json");
    let mut decoded_img = Vec::new();
    if let Some(img) = cat.image {
        decoded_img = general_purpose::STANDARD_NO_PAD.decode(img).unwrap();
    }
    let duration = start.elapsed();
    println!(
        "JSON DE: {}, duration: {:?}, img len: {}",
        cat.name,
        duration,
        decoded_img.len()
    );
}
```

Deserialization is similarly simple. We deserialize the incoming data using `serde_json` and decode the image back from base64, measuring the time that takes.

## Client and Server

Now that we have our data serialization and deserialization handled, let's write some simple server and client logic so we can try out sending our payloads over the network. 

```rust
async fn client(data: &[u8], json_data: &[u8]) -> Result<(), Box<dyn std::error::Error>> {
    println!("started in CLIENT mode");

    let mut stream = TcpStream::connect("127.0.0.1:3000").await?;
    stream.write_all(data).await?;

    let mut stream = TcpStream::connect("127.0.0.1:3001").await?;
    stream.write_all(json_data).await?;

    Ok(())
}
```

We start by implementing the client, which gets both the Cap'n Proto and JSON payloads as byte arrays and simply opens a connection to both the JSON and Cap'n Proto servers, sending the payloads across.

Next up, let's implement a simple TCP server that receives these payloads on port 3000 for Cap'n Proto and 3001 for JSON.

```rust
async fn server(port: u16) -> Result<(), Box<dyn std::error::Error>> {
    println!("started in SERVER mode");
    let listener = TcpListener::bind(format!("127.0.0.1:{}", port)).await?;

    println!("server running at 127.0.0.1:{}", port);

    loop {
        match listener.accept().await {
            Ok((mut socket, addr)) => {
                tokio::spawn(async move {
                    println!("accepted connection from: {}", addr);
                    let mut buf = [0; 1024];
                    let mut sum = 0;
                    let mut full_msg: Vec<u8> = Vec::new();

                    loop {
                        let n = match socket.read(&mut buf).await {
                            Ok(n) => {
                                if n == 0 {
                                    println!(
                                        "read {:?} bytes, msg size: {:?}",
                                        sum,
                                        full_msg.len()
                                    );
                                    match port {
                                        3000 => deserialize_cpnproto(&full_msg),
                                        3001 => deserialize_json(&full_msg),
                                        _ => unreachable!(),
                                    };
                                    return;
                                } else {
                                    full_msg.extend(buf[0..n].iter());
                                    n
                                }
                            }
                            Err(e) => {
                                println!("error reading from socket: {}", e);
                                return;
                            }
                        };
                        sum += n;
                    }
                });
            }
            Err(e) => println!("error on client connection: {}", e),
        }
    }
}
```

We'll provide the port on which to start the server and depending on that, we either deserialize the incoming payload as Cap'n Proto, or JSON. To achieve this, we start a TCP listener for the port and wait for incoming connections. Then, for each incoming connection, we start an asynchronous task, which reads the whole message and deserializes it in the correct format depending on the given port. 

There's not too much interesting stuff in this part of the code. We simply read 1024 byte chunks from the socket into a buffer and concatenate it to form the full message, which once we stop receiving more data, we deserialize.

## Putting it all together

Now we can put it all together in `main`.

```rust
mod cats_capnp;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let img = fs::read("./minka.jpg").expect("can load image");

    let args: Vec<String> = env::args().collect();
    if args.len() >= 2 {
        match &args[1][..] {
            "c" => return client(&build_cpnproto_msg(&img), &build_msg_json(&img)).await,
            "sc" => return server(3000).await,
            "sj" => return server(3001).await,
            _ => (),
        }
    }

    Ok(())
}
```

We first read the image of my wonderful cat we embedded in our data model from disk and then do a little command line argument parsing, to figure out whether we should start in client, Cap'n Proto Server, or JSON Server mode.

That's it.

Now we can run our code using the following commands and test it:

* cargo run -- sj - runs the JSON server
* cargo run -- sc - runs the Cap'n Proto server
* cargo run -- c - runs the client that sends data to the servers

Or we can build a Release binary using `cargo build --release` and run it like this:

JSON Server:

```bash
./target/release/rust-capnproto-example sj

started in SERVER mode
server running at 127.0.0.1:3001
accepted connection from: 127.0.0.1:51228
read 5135025 bytes, msg size: 5135025
JSON DE: Minka, duration: 7.354708ms, img len: 3850802
```

Cap'n Proto Server:

```bash
./target/release/rust-capnproto-example sc

started in SERVER mode
server running at 127.0.0.1:3000

accepted connection from: 127.0.0.1:51284
read 3851224 bytes, msg size: 3851224
CAPNP DE: Minka duration: 488.5µs
```

Client:

```bash
./target/release/rust-capnproto-example c


CAPNP SE: duration: 615.792µs, len: 3851224
JSON SE: duration: 4.697292ms, len: 5135025
started in CLIENT mode
```

Nice, it works! And we can also see some interesting differences in payload size and execution speed.

These measurements are *not* reliable due to anything that'll apply to micro-benchmarks and the fact that this isn't even a benchmark, but a random measurement of one singular run and the only reason I'm putting them here is to provide a general, rough, orders-of-magnitude-esque foundation for comparison.

If you're interested in actual benchmarks, you can check out [serdebench](https://github.com/llogiq/serdebench), which includes Cap'n Proto as well.

The full code can be found [here](https://github.com/zupzup/rust-capnproto-example).

## Conclusion

In this post we checked out how we can use the Cap'n Proto serialization protocol within Rust in a basic way and compared it to using JSON with Serde in terms of ergonomics and payload size.

From my perspective, without any in-depth previous experience with Protocol Buffers and such, using Cap'n Proto turned out to be quite approachable. Both the schema as well as the generated code seem reasonable the `capnp` crate seems to have great ergonomics as well.

I can definitely see myself using Cap'n Proto in one of my future Rust projects.

#### Resources

* [Cap'n Proto](https://capnproto.org/)
* [Code Example](https://github.com/zupzup/rust-capnproto-example)

