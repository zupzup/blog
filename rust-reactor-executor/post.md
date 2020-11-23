In a [previous post](https://zupzup.org/epoll-with-rust/) we looked at how to implement a simple non-blocking I/O system in Rust on Linux using epoll.

This time around, we'll go one step further, building a reactor/executor system, where we can register callbacks for asynchronous I/O processing.

If you're asking yourself at this point why this is interesting, or useful, I would refer to Richard Feynman's famous quote:

*"What I cannot create, I do not understand."*

Of course all of this has been implemented already and we could just use async/await and one of the many great async run times in Rust, but that doesn't mean we have any idea what goes on under the hood.

The end goal of this series is to build and hence understand a very basic async runtime, which supports Rust's async/await and can run Rust Futures.

But we're not there yet and the goal in this post is to create a reactor/executor system, which will enable us to register callbacks, which are executed every time a async I/O event happens.

The example, as in the above mentioned post about epoll, will be a very simplistic HTTP server.

Let's get started!

## Setup and previous Poll implementation

First, let's look at the dependencies we'll need.

```toml
[dependencies]
libc = "0.2"
rand = { version = "0.7.3", default-features = false, features = ["std"] }
lazy_static = "1.4"
```

As in the epoll example, we will use `libc` to make the system calls we need. This time around, we'll use the `rand` crate to create random event ids and we'll use `lazy_static` to make the reactor and executor into shared globals, simplifying the flow of the application.

We'll start by building the `Poll` primitive, which will enable us to register read and write I/O interest for file descriptors. For this purpose, we create a `Registry`, which memorizes the registered interests for each file descriptor and provides a public interface to modify those.

Let's start with the `Poll` primitive though, which we'll mostly take from the [previous example](https://zupzup.org/epoll-with-rust/):

```rust
type EventId = usize;


const READ_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLIN;
const WRITE_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLOUT;

macro_rules! syscall {
    ($fn: ident ( $($arg: expr),* $(,)* ) ) => {{
        let res = unsafe { libc::$fn($($arg, )*) };
        if res == -1 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(res)
        }
    }};
}

pub struct Poll {
    epoll_fd: RawFd,
}
```

We define an `EventId`, which identifies all registered events. Then we define the `syscall` macro, to conveniently be able to execute system calls, returning an error, if there is one.

Implementing `Poll` is rather straight forward as well and purely based on the implementation in the epoll post mentioned above

```rust
impl Poll {
    pub fn new() -> Self {
        let epoll_fd = syscall!(epoll_create1(0)).expect("can create epoll");
        if let Ok(flags) = syscall!(fcntl(epoll_fd, libc::F_GETFD)) {
            let _ = syscall!(fcntl(epoll_fd, libc::F_SETFD, flags | libc::FD_CLOEXEC));
        }
        Self { epoll_fd }
    }

    pub fn get_registry(&self) -> Registry {
        Registry::new(self.epoll_fd)
    }

    pub fn poll(&self, events: &mut Vec<libc::epoll_event>) {
        events.clear();
        let res = match syscall!(epoll_wait(
            self.epoll_fd,
            events.as_mut_ptr() as *mut libc::epoll_event,
            1024,
            1000 as libc::c_int,
        )) {
            Ok(v) => v,
            Err(e) => panic!("error during epoll wait: {}", e),
        };

        // safe  as long as the kernel does nothing wrong - copied from mio
        unsafe { events.set_len(res as usize) };
    }
}
```

In `new` we create the epoll queue and set it in our `Poll` struct and in `poll` we simply execute the event loop in the same way as before.

However, the `get_registry` function is new, as is the whole concept of a `Registry`.

```rust
pub struct Registry {
    epoll_fd: RawFd,
    io_sources: HashMap<RawFd, HashSet<Interest>>,
}
```

This registry is where we will store registered interests to I/O events. The registry consists of the file descriptor of the above created epoll queue and of a `HashMap` from a file descriptor to a set of interests. On every file descriptor, it's possible to set read, write or read + write interests and this is the place where we store which are set at any point in time.

Let's look at the implementation:

```rust
#[derive(PartialEq, Hash, Eq)]
pub enum Interest {
    READ,
    WRITE,
}

impl Registry {
    pub fn new(epoll_fd: RawFd) -> Self {
        Self {
            epoll_fd,
            io_sources: HashMap::new(),
        }
    }

    pub fn register_read(&mut self, fd: RawFd, event_id: EventId) -> io::Result<()> {
        let interests = self.io_sources.entry(fd).or_insert(HashSet::new());

        if interests.is_empty() {
            syscall!(epoll_ctl(
                self.epoll_fd,
                libc::EPOLL_CTL_ADD,
                fd,
                &mut read_event(event_id)
            ))?;
        } else {
            syscall!(epoll_ctl(
                self.epoll_fd,
                libc::EPOLL_CTL_MOD,
                fd,
                &mut read_event(event_id)
            ))?;
        }

        interests.clear();
        interests.insert(Interest::READ);

        Ok(())
    }

    pub fn register_write(&mut self, fd: RawFd, event_id: EventId) -> io::Result<()> {
        let interests = self.io_sources.entry(fd).or_insert(HashSet::new());

        if interests.is_empty() {
            syscall!(epoll_ctl(
                self.epoll_fd,
                libc::EPOLL_CTL_ADD,
                fd,
                &mut write_event(event_id)
            ))?;
        } else {
            syscall!(epoll_ctl(
                self.epoll_fd,
                libc::EPOLL_CTL_MOD,
                fd,
                &mut write_event(event_id)
            ))?;
        }

        interests.clear();
        interests.insert(Interest::WRITE);

        Ok(())
    }

    pub fn remove_interests(&mut self, fd: RawFd) -> io::Result<()> {
        self.io_sources.remove(&fd);
        syscall!(epoll_ctl(
            self.epoll_fd,
            libc::EPOLL_CTL_DEL,
            fd,
            std::ptr::null_mut()
        ))?;
        close(fd);

        Ok(())
    }
}
```

Alright, what's happening here? We provide methods for registering read and write interests and in each case, if there was no interest before, we simply register interest in the requested type of event with an `epoll_ctl` syscall.

If there was an entry before, we need to use the `EPOLL_CTL_MOD` flag instead of ADD, but otherwise it's the same. In any case, we clear the previous interests, setting the new one.

In a real world implementation it would be possible to have both read and write interests registered at the same time, but in our case, we'll only ever have on of these interests active to simplify things.

We also provide a way for removing any interests and closing a file descriptor, once we're done with it.

## The reactor

Now that we have our `Poll` abstraction with it's own registry, we're able to use it within the reactor part of our architecture.

The reactor instantiates a `Poll` together with it's registry and, in a separate thread, where the event loop is executed. When new events come in, we'll send it over a channel to the executor.

But let's go one step at a time:

```rust
pub struct Reactor {
    pub registry: Option<Registry>,
}
```

The `Reactor` is a simple struct with an optional registry inside. The registry isn't really "optional", the `Option` is just a mechanism so we can lazily initialize the reactor.

The implementation is pretty simple. We provide methods for registering and unregistering read and write interest from the outside.

The interesting part is within the `run` function though:

```rust
impl Reactor {
    pub fn new() -> Self {
        Self { registry: None }
    }

    pub fn run(&mut self, sender: Sender<EventId>) {
        let poller = Poll::new();
        let registry = poller.get_registry();

        self.registry = Some(registry);

        std::thread::spawn(move || {
            let mut events: Vec<libc::epoll_event> = Vec::with_capacity(1024);

            loop {
                poller.poll(&mut events);

                for e in &events {
                    sender.send(e.u64 as EventId).expect("channel works");
                }
            }
        });
    }

    pub fn read_interest(&mut self, fd: RawFd, event_id: EventId) -> io::Result<()> {
        self.registry
            .as_mut()
            .expect("registry is set")
            .register_read(fd, event_id)
    }

    pub fn write_interest(&mut self, fd: RawFd, event_id: EventId) -> io::Result<()> {
        self.registry
            .as_mut()
            .expect("registry is set")
            .register_write(fd, event_id)
    }

    pub fn close(&mut self, fd: RawFd) -> io::Result<()> {
        self.registry
            .as_mut()
            .expect("registry is set")
            .remove_interests(fd)
    }
}
```

Once we call `.run()` on a reactor, we instantiate a poller and a registry, saving the registry inside of the reactor.

Then, we start a new thread and, within that thread, call `poller.poll()`, which executes the event loop, enabling us to send the produced events to the `sender` part of the channel, which is passed into `run`. The receiver part waits for the event in the `executor`.

## The executor

The executor itself is also quite simple. In this example, we'll provide two modes of execution.

* `await_once`
* `await_keep`

Where `await_once` registers a callback function to be called ONCE, for the given `eventId` and `await_keep` registers a callback function, which is executed every time there is an event on the given `eventId`.

This isn't strictly necessary, but makes things a bit easier for us when putting everything together.

Let's look at an implementation:

```rust
pub struct Executor {
    event_map: HashMap<EventId, Box<dyn FnMut(&mut Self) + Sync + Send + 'static>>,
    event_map_once: HashMap<EventId, Box<dyn FnOnce(&mut Self) + Sync + Send + 'static>>,
}

impl Executor {
    pub fn new() -> Self {
        Self {
            event_map: HashMap::new(),
            event_map_once: HashMap::new(),
        }
    }

    pub fn await_once(
        &mut self,
        event_id: EventId,
        fun: impl FnOnce(&mut Self) + Sync + Send + 'static,
    ) {
        self.event_map_once.insert(event_id, Box::new(fun));
    }

    pub fn await_keep(
        &mut self,
        event_id: EventId,
        fun: impl FnMut(&mut Self) + Sync + Send + 'static,
    ) {
        self.event_map.insert(event_id, Box::new(fun));
    }

    pub fn run(&mut self, event_id: EventId) {
        if let Some(mut fun) = self.event_map.remove(&event_id) {
            fun(self);
            self.event_map.insert(event_id, fun);
        } else {
            if let Some(fun) = self.event_map_once.remove(&event_id) {
                fun(self);
            }
        }
    }
}
```

The executor holds two maps from `EventId` to the above mentioned callback functions. The difference between `once` and `keep` is that the former holds `FnOnce` functions, and the latter `FnMut` functions. This is where this approach makes it easier for us.

In the case of a new incoming client connection, we'll register a read interest for reading the request, read from the socket and then move on to handle the request and writing back the response. In any case, we'll have a read callback and a write callback for the same event id.

This is not a particularly elegant approach and I'm sure this can be handled more efficiently and ergonomically, but for our simple use-case this works just fine.

The `run` function of the executor will be called within an endless loop from the outside based on the events coming in on the shared channel.

In `run`, we check the `keep` map first and then the `once` map, executing the callback function with the executor as a parameter. We need the executor in here to be able to re-register callbacks from within a callback.

Because this general stage of asynchronous computation is somewhat of a middle-step from a bare event loop to an async runtime, which we'll look at in a future post, the design here might seem a bit hackish, but it still shows off some of the parts necessary to build a simple mechanism, which enables us to build a simple async I/O based application.

## Putting it all together

OK, let's put this together. First, we define some globals for the `Reactor`, `Executor` and for a map, which holds the request state of incoming client requests.


```rust
lazy_static! {
    static ref EXECUTOR: Mutex<executor::Executor> = Mutex::new(executor::Executor::new());
    static ref REACTOR: Mutex<reactor::Reactor> = Mutex::new(reactor::Reactor::new());
    static ref CONTEXTS: Mutex<HashMap<EventId, RequestContext>> = Mutex::new(HashMap::new());
}

#[derive(Debug)]
pub struct RequestContext {
    pub stream: TcpStream,
    pub content_length: usize,
    pub buf: Vec<u8>,
}

const HTTP_RESP: &[u8] = br#"HTTP/1.1 200 OK
content-type: text/html
content-length: 5

Hello"#;
```

We're wrapping these globals inside of Mutexes as well, since we might be using them from different threads concurrently.

We also define, as in the last example, a hard-coded HTTP response.

If you check the `RequestContext` implementation of the [previous post](https://zupzup.org/epoll-with-rust/), you'll notice that we had a `read_cb` and `write_cb` method in there, handling the respective read and write parts of the incoming requests. We'll do the same here:

```rust
impl RequestContext {
    fn new(stream: TcpStream) -> Self {
        Self {
            stream,
            buf: Vec::new(),
            content_length: 0,
        }
    }

    fn read_cb(&mut self, event_id: EventId, exec: &mut executor::Executor) -> io::Result<()> {
        let mut buf = [0u8; 4096];
        match self.stream.read(&mut buf) {
            Ok(_) => {
                if let Ok(data) = std::str::from_utf8(&buf) {
                    self.parse_and_set_content_length(data);
                }
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {}
            Err(e) => {
                return Err(e);
            }
        };
        self.buf.extend_from_slice(&buf);
        if self.buf.len() >= self.content_length {
            REACTOR
                .lock()
                .expect("can get reactor lock")
                .write_interest(self.stream.as_raw_fd(), event_id)
                .expect("can set write interest");

            write_cb(exec, event_id);
        } else {
            REACTOR
                .lock()
                .expect("can get reactor lock")
                .read_interest(self.stream.as_raw_fd(), event_id)
                .expect("can set write interest");
            read_cb(exec, event_id);
        }
        Ok(())
    }

    fn parse_and_set_content_length(&mut self, data: &str) {
        if data.contains("HTTP") {
            if let Some(content_length) = data
                .lines()
                .find(|l| l.to_lowercase().starts_with("content-length: "))
            {
                if let Some(len) = content_length
                    .to_lowercase()
                    .strip_prefix("content-length: ")
                {
                    self.content_length = len.parse::<usize>().expect("content-length is valid");
                    println!("set content length: {} bytes", self.content_length);
                }
            }
        }
    }

    fn write_cb(&mut self, event_id: EventId) -> io::Result<()> {
        println!("in write event of stream with event id: {}", event_id);
        match self.stream.write(HTTP_RESP) {
            Ok(_) => println!("answered from request {}", event_id),
            Err(e) => eprintln!("could not answer to request {}, {}", event_id, e),
        };
        self.stream
            .shutdown(std::net::Shutdown::Both)
            .expect("can close a stream");

        REACTOR
            .lock()
            .expect("can get reactor lock")
            .close(self.stream.as_raw_fd())
            .expect("can close fd and clean up reactor");

        Ok(())
    }
}
```

The implementation is almost identical, with the difference, that we don't deal with a `Poller` anymore, but with a `Reactor`. But same as last time, we read every time we're notified that we can read new data, until the specified content length is reached, at which point we register a write interest.

In each case, some `read_cb` and `write_cb` functions are called from outside `RequestContext`, let's look at those:

```rust
fn read_cb(exec: &mut executor::Executor, event_id: EventId) {
    exec.await_once(event_id, move |write_exec| {
        if let Some(ctx) = CONTEXTS
            .lock()
            .expect("can lock request_contexts")
            .get_mut(&event_id)
        {
            ctx.read_cb(event_id, write_exec)
                .expect("read callback works");
        }
    });
}

fn write_cb(exec: &mut executor::Executor, event_id: EventId) {
    exec.await_once(event_id, move |_| {
        if let Some(ctx) = CONTEXTS
            .lock()
            .expect("can lock request_contexts")
            .get_mut(&event_id)
        {
            ctx.write_cb(event_id).expect("write callback works");
        }
        CONTEXTS
            .lock()
            .expect("can lock request contexts")
            .remove(&event_id);
    });
}
```

In `read_cb`, we register a new `await_once` callback on the given event id, which calls the `read_cb` method of the corresponding request context.

The same happens in the `write_cb` function. In both cases we get passed in our `Executor`, so we can register new callbacks.

All of this is triggered from within the callback of the TcpListener, which happens every time we're able to `accept` a new connection:

```rust
fn listener_cb(listener: TcpListener, event_id: EventId) {
    let mut exec_lock = EXECUTOR.lock().expect("can get executor lock");
    exec_lock.await_keep(event_id, move |exec| {
        match listener.accept() {
            Ok((stream, addr)) => {
                let event_id: EventId = random();
                stream.set_nonblocking(true).expect("nonblocking works");
                println!(
                    "new client: {}, event_id: {}, raw fd: {}",
                    addr,
                    event_id,
                    stream.as_raw_fd()
                );
                REACTOR
                    .lock()
                    .expect("can get reactor lock")
                    .read_interest(stream.as_raw_fd(), event_id)
                    .expect("can set read interest");
                CONTEXTS
                    .lock()
                    .expect("can lock request contests")
                    .insert(event_id, RequestContext::new(stream));
                read_cb(exec, event_id);
            }
            Err(e) => eprintln!("couldn't accept: {}", e),
        };
        REACTOR
            .lock()
            .expect("can get reactor lock")
            .read_interest(listener.as_raw_fd(), event_id)
            .expect("re-register works");
    });
    drop(exec_lock);
}
```

Every time we're notified that our server socket is ready to read, we accept the new connection, setting the returned `TcpStream` to `nonblocking` mode. Then, we register a read interest on the file descriptor of that incoming TCP stream, put the stream and the (randomly generated) event id in our shared request context map and finally call the `read_cb`, which registers the callback function, which will be called once the stream is ready to be read.

Also, we need to re-register the read interest for the TCPListener every time we accept a connection.

The final part is to put it all together in `main` and to test if it works:


```rust
fn main() -> io::Result<()> {
    let listener_event_id = 100;
    let listener = TcpListener::bind("127.0.0.1:8000")?;
    listener.set_nonblocking(true)?;
    let listener_fd = listener.as_raw_fd();

    let (sender, receiver) = channel();

    match REACTOR.lock() {
        Ok(mut re) => re.run(sender),
        Err(e) => panic!("error running reactor, {}", e),
    };

    REACTOR
        .lock()
        .expect("can get reactor lock")
        .read_interest(listener_fd, listener_event_id)?;

    listener_cb(listener, listener_event_id);

    while let Ok(event_id) = receiver.recv() {
        EXECUTOR
            .lock()
            .expect("can get an executor lock")
            .run(event_id);
    }

    Ok(())
}
```

We create a TcpListener, listening on port `8000`, create a shared `channel` and run the `Reactor` with the send part of that channel.

Then we immediately register a read interest on the server event id and register the server callback function.

At the end, we wait for incoming events from the `Reactor` and call `run` for every incoming event.

Let's try it by starting the application with `cargo run` and making an HTTP request using `cURL`.

```bash
while true; do curl --location --request POST 'http://localhost:8000/upload' \--form 'file=@/home/somewhere/some_image.png' -w ' Total: %{time_total}' && echo '\n'; done;
```

The log shows our incoming request and our handling of it:

```bash
...
new client: 127.0.0.1:55976, event_id: 15205027030263948942, raw fd: 5
set content length: 480721 bytes
in write event of stream with event id: 15205027030263948942
answered from request 15205027030263948942
...
```

You can also start lots of other such loops and see how we can handle all of the requests concurrently.

That's it - the full code can be found [here](https://github.com/zupzup/rust-reactor-executor-example)

## Conclusion

Another step complete towards our goal of understanding how asynchronous I/O works in Rust. The next step will be to build a simple async runtime, which can run Rust Futures.

This simplistic reactor/executor stage is a bit wonky and my architecture is highly inefficient and un-ergonomic, but it does work and we're able to fully asynchronously handle incoming HTTP requests. So yay! :)

Below you can find some useful resources on async basics and async I/O I used in my research for this blog post.

#### Resources

* [Code Example](https://github.com/zupzup/rust-reactor-executor-example)
* [Exploring Async Basics](https://cfsamson.github.io/book-exploring-async-basics/introduction.html)
* [Async I/O Rust Library](https://github.com/smol-rs/async-io)
