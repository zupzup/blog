TBD Intro

Example we're going to build etc.

Not a fair comparison, not to bash, but to actually compare and see how ergonomics look and roughly how the orders of magnitude look like.

## Setup

```toml
[dependencies]
capnp = "=0.17.2"
tokio = { version = "=1.29.1", features = ["full"] }
serde = {version = "=1.0", features = ["derive"] }
serde_json = "=1.0"
base64 = "=0.21.2"

[build-dependencies]
capnpc = "=0.17.2"

```


## TBD Schema and build.rs

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

TBD cargo build, check out `src/cats_capnp.rs`. Import using `mod cats_capnp`.

## TBD JSON

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

## TBD Cap'n Proto

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

## TBD Client/Server

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
```rust
```
```rust
```
```rust
```
```rust
```
## TBD all together

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

TBD testing with time/size


The full code can be found [here](https://github.com/zupzup/rust-capnproto-example).

## Conclusion

TBD

#### Resources

* [Cap'n Proto](https://capnproto.org/)
* [Code Example](https://github.com/zupzup/rust-capnproto-example)

