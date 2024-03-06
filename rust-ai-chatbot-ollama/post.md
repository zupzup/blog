Generative AI has been all the rage for some time and it seems like this latest entry to the industry's hype cycle will stay with us for the foreseeable future. Between that and the fact that I was looking for an interesting example for a coding dojo at work, I thought it would be interesting to build something that leverages one of the open source LLMs in Rust. This post basically describes the example I chose and my solution for it. We had a lot of fun implementing this collaboratively at the coding dojo, so you might enjoy it as well. :)

The [Ollama](https://ollama.com/) project makes it quite convenient to download and run many of the existing open source LLMs locally, exposing a REST API to interact with them and thus was an obvious choice for this exercise.

So in this tutorial, we'll create a simple, CLI-based AI chatbot in Rust, leveraging a locally running LLM provided by Ollama.

Let's get started!

## Setup

As mentioned above, we'll use Ollama to run an LLM locally. You can download and install Ollama as described [here](https://ollama.com/download). Then, locally, you can start it by running `ollama run llama2`. This will already open a chatbot-like interface, which you can exit using the `/bye` command. At this point, you can also send HTTP requests to the model, e.g. using cURL:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama2",
  "messages": [
    { "role": "user", "content": "hello AI, what's happenin'?" }
  ]
}'
```

Notice the `user` role in the request. The LLM will always answer in the `assistant` role. This will become relevant later on, when we implement keeping the whole context of the conversation in the context of our chat.

To set up the Rust project, we use cargo:

```bash

cargo new rust-ai-chatbox-example
cd rust-ai-chatbox-example
```

And then we can edit the `Cargo.toml` file and add some dependencies we'll need.

```toml
[dependencies]
reqwest = { version = "0.11.24", features = ["json"] }
serde = { version = "1.0.197", features = ["derive"] }
serde_json = "1.0.114"
tokio = { version = "1.36.0", features = ["full"] }
```

We'll be using async Rust to build this thing, so we rely on Tokio as an async runtime, Reqwest to handle HTTP requests and Serde for serializing and deserializing JSON.

That's it, let's get coding!

## Implementing the CLI

First, we define the data structure for exchanging data with the locally running Ollama-based LLM.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Response {
    message: Message,
    done: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Message {
    role: String,
    content: String,
}
```

We'll read directly from `stdin` and write to `stdout`. Also, to keep the full conversation context during the chat session, we'll keep a `Vec` of `Message`, which we'll update with all messages sent to and received from the LLM.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let messages: Vec<Message> = vec![];
    let client = reqwest::Client::new();

    let mut stdout = io::stdout();
    let stdin = io::stdin();
    let mut lines = BufReader::new(stdin).lines();
```

Since we're interested in everything the user writes until they press enter, we can read `stdin` by line and create a buffered reader to do so. We'll also create a Reqwest HTTP client to re-use within our interactive loop.

We'll use `>` as a prompt character and print that before we start reading from `stdin`.

```rust
    stdout.write_all(b"\n> ").await?;
    stdout.flush().await?;
```

Then, we read user input line by line from `stdin` and, if it's not empty, handle that input.

```rust
    while let Some(line) = lines.next_line().await? {
        if !line.is_empty() {
            messages.push(Message {
                role: String::from("user"),
                content: line,
            });
```

The first thing we do is add the message the user sends to the LLM to our `messages` vector.

This vector is also what we'll always send to the LLM API endpoint, since it represents the whole conversation context. If we wouldn't do that, each message to the LLM would essentially be without context and always start at the base prompt.

```rust

            let mut res = client
                .post("http://127.0.0.1:11434/api/chat")
                .json(&json!({
                    "model": "llama2",
                    "messages": &messages

                }))
                .send()
                .await?;
```

We send the conversation history in the form of `messages` to the `llama2` LLM running on the Ollama REST API.

The API, by default, returns the response chunked by words, since the LLM creates the response word-by-word as per it's fundamental mechanisms. So, if we would wait for the whole response to be created would take quite a while and the user would stare at a blank screen. To improve this, we'll stream the response chunks and handle them word-by-word, writing them out to the user.

```rust
            let mut message = String::default();
            while let Some(chunk) = res.chunk().await? {
                if let Ok(resp_part) = serde_json::from_slice::<Response>(&chunk) {
                    if !resp_part.done {
                        stdout
                            .write_all(resp_part.message.content.as_bytes())
                            .await?;
                        stdout.flush().await?;

                        message.push_str(&resp_part.message.content);
                    }
                }
            }
```

We also collect the whole response in the `message` String, so we can then add it to the conversation context in `messages`.

```rust
            if !message.is_empty() {
                messages.push(Message {
                    role: String::from("assistant"),
                    content: message,
                });
            }
        }
```

Finally, we print the `>` prompt character again for the next input.

```rust
        stdout.write_all(b"\n> ").await?;
        stdout.flush().await?;
```

That's it for our little CLI app. Let's test, if it does what we think it does!

## Testing

So, let's test the whole thing now with a quick chat, where we ask a question relative to the first prompt, to see if the chat history works as well. Let's see how much Llama2 knows about Austrian literature!

```bash
rust-ai-chatbot-ollama ❯ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.16s
     Running `target/debug/rust-ai-chatbot-ollama`

> Hey, did Elfriede Jelinek ever win the Nobel Prize for Literature?
Elfriede Jelinek has not won the Nobel Prize in Literature. Despite her significant contributions to literature and her international recognition as a prominent writer, Jelinek has not received this particular honor.
> Are you sure? I'm pretty sure she won it in 2004
I apologize, you are correct. Elfriede Jelinek was awarded the Nobel Prize in Literature in 2004. Thank you for bringing this to my attention.
> Indeed. Do you also know who won it the year after, in 2005?
Yes, the Nobel Prize in Literature was awarded to Harold Pinter in 2005.
> That's correct, awesome! Thank you
You're welcome! It's always great to learn and share knowledge. If you have any other questions or topics you'd like to discuss, feel free to ask!
```

Nice, it works! You can find the full code [here](https://github.com/zupzup/rust-ai-chatbot-ollama).

## Conclusion

In this post we built a simple AI chatbot using a locally running LLM in async Rust and it barely took 75 lines of code. Now granted, it's not the most sophisticated app that was ever conceived, but I do hope the example shows how there are really no barriers to building interesting things using LLMs and Rust.

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/rust-ai-chatbot-ollama)
* [Ollama](https://ollama.com/)
* [Tokio](https://tokio.rs/)
* [Serde](https://serde.rs/)
* [Reqwest](https://github.com/seanmonstar/reqwest)

