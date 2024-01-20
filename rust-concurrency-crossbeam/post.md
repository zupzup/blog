*This post was originally posted on the [LogRocket](https://blog.logrocket.com/concurrent-programming-rust-crossbeam/) blog on 07.12.2022 and was cross-posted here by the author.*

In this article, we’re going to look at the [Crossbeam](https://github.com/crossbeam-rs/crossbeam) crate, which provides tools for concurrent programming in Rust. The library is widely used under the hood in the Rust ecosystem by many libraries and frameworks, provided concurrent programming is within their domain.

[This](https://aturon.github.io/blog/2015/08/27/epoch/) fantastic blog post by Aaron Turon introduced Crossbeam in 2015 and offers some great insight into the challenges that arise with regard to lock-free programming with Rust — if you have the time, I definitely recommend giving it a read.

Without further ado, let’s get into concurrent programming in Rust using Crossbeam!

## Concurrent programming in Rust

The TL;DR of this is that it’s possible to build efficient lock-free data structures with Rust, but you actually need to build a memory-reclamation mechanism — similar to a garbage collector.

In the case outlined in the blog above, Turon implemented an epoch-based memory management API, which can be used as a basis to build lock-free data structures in Rust

This epoch-based memory-reclamation mechanism is also part of the library — it is [well-documented](https://docs.rs/crossbeam/latest/crossbeam/epoch/index.html) if you would like to learn more.

If you’re not too familiar with lock-free programming, you can check out this great [introduction](https://preshing.com/20120612/an-introduction-to-lock-free-programming/).

In this post, we will look at a part of Crossbeam’s API and implement some simple examples using them to showcase how they can be used and why they’re useful.

If you’re interested in checking out the whole API to go more in-depth about the different concurrent-programming tools within Crossbeam, you can also check out the [docs](https://docs.rs/crossbeam/latest/crossbeam/index.html).

Let’s get started!

## Setup

To follow along, all you need is a recent Rust installation (the latest version at the time of writing is 1.64.0).

First, create a new Rust project:

```bash
    cargo new rust-crossbeam-example
    cd rust-crossbeam-example
```

Next, edit the `Cargo.toml` file and add the dependencies you'll need:

```toml
    [dependencies]
    crossbeam = "0.8.2"
    rand = "0.8.5"
```

Crossbeam and `rand` are all we will need for the upcoming examples :)

## `AtomicCell` example

We’ll start with `AtomicCell`, a thread-safe implementation of Rust’s `Cell` (more [info](https://doc.rust-lang.org/nightly/core/cell/struct.Cell.html)).

`Cell` is a mutable memory location — as is `AtomicCell` — the difference being that it’s thread-safe, so multiple threads can access and mutate the memory location at the same time without data races.

We can test this by spawning threads — in some, we can also load and print the value in `AtomicCell`, and, in others, increment and print it. Once the threads are finished, the result always needs to be the same.

First, let’s define a `run_thread` helper, which takes an `Arc` (a thread-safe, reference-counted smart pointer) containing the `AtomicCell`. Our mutable memory location, the number of the thread, and whether it should store something should be defined.

```rust
    fn run_thread(val: Arc<AtomicCell<u32>>, num: u32, store: bool) -> thread::JoinHandle<()> {
        thread::spawn(move || {
            if store {
                val.fetch_add(1);
            }
            println!("Hello from thread {}! value: {}", num, val.load());
        })
    }
```

With that, we can now create our `AtomicCell` and initialize it with the number `12`. Then, we put it into an `Arc`, so we can share it between threads, spawn threads using `run_thread`, and then wait for the threads to finish.

```rust
    fn main() {
        // AtomicCell Example
        println!("Starting AtomicCell example...");
        let atomic_value: AtomicCell<u32> = AtomicCell::new(12);
        let arc = Arc::new(atomic_value);
    
        let mut thread_handles_ac: Vec<thread::JoinHandle<()>> = Vec::new();
        for i in 1..10 {
            thread_handles_ac.push(run_thread(arc.clone(), i, i % 2 == 0));
        }
    
        thread_handles_ac
            .into_iter()
            .for_each(|th| th.join().expect("can't join thread"));
    
        println!("value after threads finished: {}", arc.load());
        println!("AtomicCell example finished!");
    }
```

If you run this using `cargo run` (repeatedly), the result will always be the same:

```bash
    Starting AtomicCell example...
    Hello from thread 1! value: 12
    Hello from thread 4! value: 13
    Hello from thread 5! value: 13
    Hello from thread 2! value: 14
    Hello from thread 3! value: 14
    Hello from thread 8! value: 15
    Hello from thread 6! value: 16
    Hello from thread 7! value: 16
    Hello from thread 9! value: 16
    value after threads finished: 16
    AtomicCell example finished!
```

`AtomicCell` is not something you should expect to use a lot in application code, but it’s a very useful tool for building concurrent-programming primitives that deal with mutable state.

## `ArrayQueue` example

The next piece of Crossbeam’s API we’ll look at is `ArrayQueue`.

As the name suggests, this is a thread-safe queue. In particular, it’s a bounded (i.e., limited buffer), multi-producer, multi-consumer queue.

To see it in action, we will create a number of producer and consumer threads, which will push and pop values into and from the queue — at the end, we should expect consistent results.

To achieve this, we must first create two helper functions for creating producer and consumer threads:

```rust
    fn run_producer(q: Arc<ArrayQueue<u32>>, num: u32) -> thread::JoinHandle<()> {
        thread::spawn(move || {
            println!("Hello from producer thread {} - pushing...!", num);
            for _ in 0..20 {
                q.push(num).expect("pushing failed");
            }
        })
    }
    
    fn run_consumer(q: Arc<ArrayQueue<u32>>, num: u32) -> thread::JoinHandle<()> {
        thread::spawn(move || {
            println!("Hello from producer thread {} - popping!", num);
            for _ in 0..20 {
                q.pop();
            }
        })
    }
```

We pass an `ArrayQueue` packaged inside of an `Arc` into the helper and, within it, start a thread which, in a small loop, pushes to (for producers) and pops from (for consumers) the queue.

```rust
    fn main() {
        // ArrayQueue Example
        println!("---------------------------------------");
        println!("Starting ArrayQueue example...");
        let q: ArrayQueue<u32> = ArrayQueue::new(100);
        let arc_q = Arc::new(q);
    
        let mut thread_handles_aq: Vec<thread::JoinHandle<()>> = Vec::new();
    
        for i in 1..5 {
            thread_handles_aq.push(run_producer(arc_q.clone(), i));
        }
    
        for i in 1..5 {
            thread_handles_aq.push(run_consumer(arc_q.clone(), i));
        }
    
        thread_handles_aq
            .into_iter()
            .for_each(|th| th.join().expect("can't join thread"));
        println!("values in q after threads finished: {}", arc_q.len());
        println!("ArrayQueue example finished!");
    }
```

Then, after initializing the bounded queue, we put it into an `Arc`, spawn our producers and consumers, and then wait for the threads to finish.

We limit it to a buffer of 100 entries; this means that pushing to the queue can actually fail if the queue is full

If we check out the results, we can see, regardless of the ordering, that the results are consistent and the queue is actually thread-safe:

```bash
    Starting ArrayQueue example...
    Hello from producer thread 1 - pushing...!
    Hello from producer thread 4 - pushing...!
    Hello from producer thread 2 - pushing...!
    Hello from producer thread 2 - popping!
    Hello from producer thread 3 - pushing...!
    Hello from producer thread 1 - popping!
    Hello from producer thread 4 - popping!
    Hello from producer thread 3 - popping!
    values in q after threads finished: 0
    ArrayQueue example finished!
```

`ArrayQueue`, similar to `AtomicCell`, is something that will mostly be useful within abstractions for [concurrency](https://blog.logrocket.com/deep-dive-concurrency-rust-programming-language/), rather than actual application code.

If you have a use case for which you need a queue data structure that’s thread-safe, Crossbeam has you covered! There are also two other queue implementations worth noting: `deque` and `SegQueue`, which you can check out [here](https://docs.rs/crossbeam/latest/crossbeam/index.html#data-structures).

## Channel example

Channels, a concept you might know from other languages such as Go, are implemented in a multi-producer, multi-consumer manner in Crossbeam.

Channels are usually used for cross-thread communication — or, in the case of Go, cross-goroutine communication. 

For example, with channels, you can implement a CSP — communicating sequential processes for the coordination of concurrent processes.

To see Crossbeam channels in action, we will again spawn producer and consumer threads.

However, this time, the producers will send 1000 values each into the channel and the consumers will simply take values from it and process them — if there are no senders left, all threads will finish.

To achieve this, we first create two helper functions for creating the producer and consumer threads:

```rust
    fn run_producer_chan(s: Sender<u32>, num: u32) -> thread::JoinHandle<()> {
        thread::spawn(move || {
            println!("Hello from producer thread {} - pushing...!", num);
            for _ in 0..1000 {
                s.send(num).expect("send failed");
            }
        })
    }
    
    fn run_consumer_chan(r: Receiver<u32>, num: u32) -> thread::JoinHandle<()> {
        thread::spawn(move || {
            let mut i = 0;
            println!("Hello from producer thread {} - popping!", num);
            loop {
                if let Err(_) = r.recv() {
                    println!(
                        "last sender dropped - stopping consumer thread, messages received: {}",
                        i
                    );
                    break;
                }
                i += 1;
            }
        })
    }
```

Similar to the above, the producer thread simply pushes values into the channel. However, the consumer thread, in an endless loop, tries to receive from the channel and, if it gets an error (which happens if all senders are dropped), then it prints the number of processed messages for the thread.

To put this all together, we created an unbounded `channel`, getting a `Sender` and a `Receiver` back, which can be cloned and shared between threads (there are also bounded channels with a limited buffer size). We then spawned our producers and consumers and left it to be run.

We also dropped the initial `Sender` using `drop(s)`. Since we rely on the consumer threads running into the error condition, when all `Senders` are dropped and we clone the `Sender` into each thread, we need to remove the initial `Sender` reference; otherwise, the consumers will simply block forever in an endless loop.

```rust
    fn main() {
            // channel Example
        println!("---------------------------------------");
        println!("Starting channel example...");
        let (s, r) = unbounded();
    
        for i in 1..5 {
            run_producer_chan(s.clone(), i);
        }
        drop(s);
    
        for i in 1..5 {
            run_consumer_chan(r.clone(), i);
        }
    
        println!("channel example finished!");
    }
```

If we run this repeatedly, the number of messages each thread processes will vary, but the overall amount will always add up to 4000, meaning that all events sent to the channels have been processed.

```bash
    Starting channel example...
    Hello from producer thread 1 - pushing...!
    Hello from producer thread 2 - pushing...!
    Hello from producer thread 4 - pushing...!
    Hello from producer thread 3 - pushing...!
    Hello from producer thread 4 - popping!
    Hello from producer thread 2 - popping!
    Hello from producer thread 3 - popping!
    Hello from producer thread 1 - popping!
    last sender dropped - stopping consumer thread, messages received: 376
    last sender dropped - stopping consumer thread, messages received: 54
    last sender dropped - stopping consumer thread, messages received: 2199
    last sender dropped - stopping consumer thread, messages received: 1371
    channel example finished!
```

## `WaitGroup` example

Wait groups are a very useful concept for cases when you have to do some concurrent processing and must wait until it’s all finished. For example; to wait for all the data to be collected from different sources in parallel and then waiting for each request to finish, before aggregating it and moving on in the computation process.

The idea is to create a `WaitGroup` and then, for each concurrent process, clone it (internally it simply increases a counter) and wait for all WaitGroups to be dropped (i.e., the counter is back to 0). That way, in a thread-safe way, we have a guarantee that all threads have finished.

To showcase this, we’ll create a helper called `do_work`, which generates a random number, sleeps for that amount of milliseconds, then does some basic calculations, sleeps again, and finishes. 

This is just so we actually have our threads doing something, which takes a different amount of time for each thread.

```rust
    fn do_work(thread_num: i32) {
        let num = rand::thread_rng().gen_range(100..500);
        thread::sleep(std::time::Duration::from_millis(num));
        let mut sum = 0;
        for i in 0..10 {
            sum += sum + num * i;
        }
        println!(
            "thread {} calculated sum: {}, num: {}",
            thread_num, sum, num
        );
        thread::sleep(std::time::Duration::from_millis(num));
    }
```

Then, with our `WaitGroup` created, we create a number of threads — 50 in this case — and clone the `WaitGroup` for each of the threads, dropping it inside the thread again.

Afterward, we wait for the `WaitGroup`, which will block until all `WaitGroup` clones have been dropped — and thus all threads have finished.

```rust
    fn main() {
        // WaitGroup Example
        println!("---------------------------------------");
        println!("Starting WaitGroup example...");
        let wg = WaitGroup::new();
    
        for i in 0..50 {
            let wg_clone = wg.clone();
            thread::spawn(move || {
                do_work(i);
                drop(wg_clone);
            });
        }
    
        println!("waiting for all threads to finish...!");
        wg.wait();
        println!("all threads finished!");
        println!("WaitGroup example finished!");
    }
```

If we run this, we can see that our code actually consistently waits for all of our 50 threads, no matter how long they take to complete.

```bash
    Starting WaitGroup example...
    waiting for all threads to finish...!
    thread 41 calculated sum: 114469, num: 113
    thread 31 calculated sum: 116495, num: 115
    thread 20 calculated sum: 119534, num: 118
    thread 18 calculated sum: 126625, num: 125
    thread 37 calculated sum: 144859, num: 143
    thread 47 calculated sum: 147898, num: 146
    thread 42 calculated sum: 170184, num: 168
    thread 11 calculated sum: 185379, num: 183
    thread 17 calculated sum: 186392, num: 184
    thread 19 calculated sum: 188418, num: 186
    thread 35 calculated sum: 195509, num: 193
    thread 34 calculated sum: 197535, num: 195
    thread 4 calculated sum: 200574, num: 198
    thread 39 calculated sum: 202600, num: 200
    thread 25 calculated sum: 215769, num: 213
    thread 6 calculated sum: 223873, num: 221
    thread 22 calculated sum: 227925, num: 225
    thread 12 calculated sum: 256289, num: 253
    thread 49 calculated sum: 265406, num: 262
    thread 30 calculated sum: 267432, num: 264
    thread 43 calculated sum: 271484, num: 268
    thread 27 calculated sum: 283640, num: 280
    thread 23 calculated sum: 303900, num: 300
    thread 48 calculated sum: 304913, num: 301
    thread 14 calculated sum: 306939, num: 303
    thread 0 calculated sum: 309978, num: 306
    thread 5 calculated sum: 324160, num: 320
    thread 13 calculated sum: 333277, num: 329
    thread 40 calculated sum: 338342, num: 334
    thread 28 calculated sum: 346446, num: 342
    thread 46 calculated sum: 358602, num: 354
    thread 29 calculated sum: 362654, num: 358
    thread 1 calculated sum: 368732, num: 364
    thread 15 calculated sum: 368732, num: 364
    thread 38 calculated sum: 386966, num: 382
    thread 24 calculated sum: 419382, num: 414
    thread 44 calculated sum: 430525, num: 425
    thread 45 calculated sum: 430525, num: 425
    thread 8 calculated sum: 433564, num: 428
    thread 32 calculated sum: 433564, num: 428
    thread 16 calculated sum: 442681, num: 437
    thread 2 calculated sum: 443694, num: 438
    thread 26 calculated sum: 444707, num: 439
    thread 36 calculated sum: 454837, num: 449
    thread 21 calculated sum: 456863, num: 451
    thread 7 calculated sum: 458889, num: 453
    thread 33 calculated sum: 459902, num: 454
    thread 3 calculated sum: 488266, num: 482
    thread 10 calculated sum: 497383, num: 491
    thread 9 calculated sum: 505487, num: 499
    all threads finished!
    WaitGroup example finished!
```

That’s it! The full example code can be found on [GitHub](https://github.com/zupzup/rust-crossbeam-example).

## Conclusion

In this article, we looked at some parts of the powerful Crossbeam library, which is an absolute staple in Rust when it comes to [concurrent](https://blog.logrocket.com/deep-dive-concurrency-rust-programming-language/) programming.

Crossbeam has more useful tools and abstractions to offer, and if you’re interested in checking them out, feel free to scour the fantastic [docs](https://docs.rs/crossbeam/latest/crossbeam/index.html).

The concepts behind Crossbeam with regard to lock-free programming and its implications in combination with Rust are quite interesting and definitely an area worth exploring deeper to get a fundamental understanding of the effects of lock-free programming and where it’s useful.

