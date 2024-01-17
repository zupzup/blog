*This post was originally posted on the [LogRocket](https://blog.logrocket.com/file-upload-and-download-in-rust/) blog on 09.09.2020 and was cross-posted here by the author.*

Many web applications, especially user-facing ones, provide an interface for uploading files. This can range from simple files, such as an avatar picture, to complex items such as ultra-secure, cryptographically signed contracts.

In any case, for many web services, this is a must-have - just as critical as the ability to download or access these files again once they are uploaded.

In this guide, we’ll demonstrate how to implement file upload and download in a Rust web application. We’ll use the [warp](https://github.com/seanmonstar/warp) web framework, but the basics will mostly apply to other async web frameworks.

There’s a nifty crate for dealing with async multipart streams called [mpart-async](https://crates.io/crates/mpart-async), which streams incoming files instead of loading them into memory completely, as warp does at this time (there is an [issue](https://github.com/seanmonstar/warp/issues/323) for this).

However, for the purpose of this tutorial, we’ll use warp’s built-in method for handling multipart requests. If your use case involves dealing with huge files you need to stream through or validate before loading into memory, the aforementioned `mpart-async` crate is a sensible way to go for now.

We’ll build an example web application that enables users to upload `.pdf` and `.png` files up to 5 MB. Each file will be placed in the local `./files` folder with a randomly generated name and made available for download using the `GET /files/$filename` endpoint.

## Setup

To follow along, all you need is a reasonably recent Rust installation (1.39+) and a tool to send HTTP requests, such as cURL.

First, create a new Rust project.

```bash
cargo new rust-upload-download-example
cd rust-upload-download-example
```

Next, edit the Cargo.toml file and add the dependencies you’ll need.

```toml
[dependencies]
tokio = { version = "0.2.21", features = ["macros", "rt-threaded", "fs"] }
warp = "0.2.3"
uuid = { version = "0.8", features = ["v4"] }
futures = { version = "=0.3.5", default-features = false }
bytes = "0.5.6"
```

We’re using warp to build the web service, which uses Tokio underneath. The other dependencies are used to handle the file uploads with warp. With `uuid`, we’ll create unique names for the uploaded files and the futures and bytes crates will help us deal with the incoming file stream.

## Web service

Let’s start by creating a basic Warp web application that lets users download files from a local folder.

```rust
#[tokio::main]
async fn main() {
    let download_route = warp::path("files").and(warp::fs::dir("./files/"));

    let router = download_route.recover(handle_rejection);
    println!("Server started at localhost:8080");
    warp::serve(router).run(([0, 0, 0, 0], 8080)).await;
}

async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
    let (code, message) = if err.is_not_found() {
        (StatusCode::NOT_FOUND, "Not Found".to_string())
    } else if err.find::<warp::reject::PayloadTooLarge>().is_some() {
        (StatusCode::BAD_REQUEST, "Payload too large".to_string())
    } else {
        eprintln!("unhandled error: {:?}", err);
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            "Internal Server Error".to_string(),
        )
    };

    Ok(warp::reply::with_status(message, code))
}
```

In this code snippet, we define a `download_route` at `GET /files`, which, using Warp’s `fs::dir` filter, serves files from the given path. That’s all we have to do to implement file downloads.

Of course, depending on your specific use case and the size of the files you’re dealing with — or if you have security concerns — the logic for downloading files will be more complex than this. But it’s adequate for our example.

## Uploading files

Next, let’s look at the `upload` route definition.

```rust
// in main
let upload_route = warp::path("upload")
    .and(warp::post())
    .and(warp::multipart::form().max_length(5_000_000))
    .and_then(upload);

let router = upload_route.or(download_route).recover(handle_rejection);
```

The upload route is a `POST` endpoint. We use Warp’s `multipart::form()` filter to pass through multipart requests.

We can also define a max length here. You might have noticed in the `handle_rejection` error handling method that we explicitly handle the `PayloadTooLarge` error. This error is triggered when a payload exceeds this limit.

Finally, let’s check out the `upload` handler. This is the core piece of this small application, and it’s a bit more complex, so we’re going to go over it step by step.

```rust
async fn upload(form: FormData) -> Result<impl Reply, Rejection> {
    let parts: Vec<Part> = form.try_collect().await.map_err(|e| {
        eprintln!("form error: {}", e);
        warp::reject::reject()
    })?;
```

We immediately see the `FormData` in the function signature. This is actually a `warp::multipart::FormData`, which is a stream of multipart `Part` elements. Since we’re dealing with a `futures::Stream`, we can use the `TryStreamExt` trait for some helpers. In this case, we use the `try_collect` function to gather the whole stream into a collection asynchronously, logging the error if this fails.

To understand the next part, let’s look at an example request we might send to this server using cURL.

```bash
curl --location --request POST 'http://localhost:8080/upload' \
     --header 'Content-Type: multipart/form-data' \
     --form 'file=@/home/somewhere/picture.png'
```

As you can see in the `--form` option, we define our file with the name `file`. We could also add additional parameters, such as a file name or some additional metadata.

The next step is to iterate over our `Parts` collected above to see if there is a `file` field.

```rust
    for p in parts {
        if p.name() == "file" {
            let content_type = p.content_type();
...
```

We iterate over the parts we got and, if one is called file, we’ll assume it’s a file. Now we need to make sure it has the correct content type — in this case, PDF or PNG — and process it further.

```rust
            let file_ending;
            match content_type {
                Some(file_type) => match file_type {
                    "application/pdf" => {
                        file_ending = "pdf";
                    }
                    "image/png" => {
                        file_ending = "png";
                    }
                    v => {
                        eprintln!("invalid file type found: {}", v);
                        return Err(warp::reject::reject());
                    }
                },
                None => {
                    eprintln!("file type could not be determined");
                    return Err(warp::reject::reject());
                }
            }
```

We can use the `.content_type()` method of `Part` to check the actual file type. rust-mime-sniffer is another useful crate you could use to detect the file type.

We also set the `file_ending` based on the file type, so we can append it to the file we want to create later on.

If the file is of an unknown type or doesn’t have a known type, we log the error and return.

The next step is to convert the `Part` into a byte vector we can actually write to disk.

```rust
            let value = p
                .stream()
                .try_fold(Vec::new(), |mut vec, data| {
                    vec.put(data);
                    async move { Ok(vec) }
                })
                .await
                .map_err(|e| {
                    eprintln!("reading file error: {}", e);
                    warp::reject::reject()
                })?;
```

At this point, we can use Part‘s `.stream()` or `.data()` methods to get to the underlying `bytes::Buf` containing the data.

In this example, we turn the whole part into a stream again and, using another helper from `TryStreamExt` called `.try_fold()`, concatenate all the buffers to one `Vec<u8>` with all our data.

This `try_fold` is essentially similar to any other `.reduce()` or `.fold()` function, except that it’s asynchronous. We define an initial value (an empty vector) and then add each piece of data to it.

If any of this fails, we again log the error and return.

At this point, we parsed and validated the incoming file and have the data ready to be written to disk, which is the final step.

```rust
            let file_name = format!("./files/{}.{}", Uuid::new_v4().to_string(), file_ending);
            tokio::fs::write(&file_name, value).await.map_err(|e| {
                eprint!("error writing file: {}", e);
                warp::reject::reject()
            })?;
            println!("created file: {}", file_name);
        }
    }
    Ok("success")
}
```

First, we create a randomly generated, unique file name using the `Uuid` crate and add the above-calculated `file_ending`. Then, using Tokio’s `fs::write`, which is an asynchronous equivalent to `std::fs::write`, we write the data to a file with the generated file name. If it works out, we log the file name and return a success message to the caller.

Let’s try it using the following commands.

```bash
# first, upload some png
curl --location --request POST 'http://localhost:8080/upload' \
     --header 'Content-Type: multipart/form-data' \
     --form 'file=@/home/somewhere/picture.png'

# check the logs for the file name, or go into the ./files folder
created file: ./files/7d678724-9480-489e-8a33-57e1ae5adb4d.png

# request the file and pipe it to a new png
curl http://localhost:8080/files/7d678724-9480-489e-8a33-57e1ae5adb4d.png > new_picture.png
```

It works! Fantastic. And all that in less than a hundred lines of Rust code with some very basic error handling and input validation using just a couple of crates.

You can find the full example code at [GitHub](https://github.com/zupzup/warp-upload-download-example).

## Conclusion

Efficiently and robustly handling file uploads in a web service is not an easy task, but the Rust ecosystem provides all the tools to do so, even with options to asynchronously stream files for additional speed and flexibility.

The above example is a starting point for a real-world implementation. While it’s used in production systems, there are more things to consider. That said, the fundamentals are already there in the Rust web ecosystem to create great upload and download experiences for your users.

