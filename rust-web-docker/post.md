*This post was originally posted on the [LogRocket](https://blog.logrocket.com/packaging-a-rust-web-service-using-docker/) blog on 23.06.2020 and was cross-posted here by the author.*

Since Docker ushered in a wave of container hype several years ago, it has become somewhat standard, especially in the cloud, to package web applications as Docker containers.

This approach has some nice benefits for both local development and server deployments. Once an image has been built, it doesn’t change and can be executed on any platform that has a running Docker engine with a minimal performance penalty (on Linux).

In this tutorial, we’ll demonstrate how to put a Rust web application inside a Docker container. We’ll walk through two approaches with different base images and tradeoffs.

## Basic web app

First, let’s build a very basic web application with warp to test the different docker setups.

In this case, the only dependencies you’ll need are tokio and warp itself:

```toml
[dependencies]
tokio = { version = "0.2", features = ["macros", "rt-threaded"] }
warp = "0.2"
```

You’ll also need a running web server with a `/health` endpoint to call to see if everything works.

```rust
use warp::{Filter, Rejection, Reply};

type Result<T> = std::result::Result<T, Rejection>;

#[tokio::main]
async fn main() {
    let health_route = warp::path!("health").and_then(health_handler);

    let routes = health_route.with(warp::cors().allow_any_origin());

    println!("Started server at localhost:8000");
    warp::serve(routes).run(([0, 0, 0, 0], 8000)).await;
}

async fn health_handler() -> Result<impl Reply> {
    Ok("OK")
}
```

The code for the basic web app isn’t particularly exciting. However, it’s important to note the criticality of the `0.0.0.0` when binding the server to an IP and port. Using `127.0.0.1` or localhost here won’t work from inside docker.

Running cargo run with this in place will start a local server on http://localhost:8000 with a /health endpoint, which returns OK when it’s called.

## Docker image (Debian)

Now it’s time to put the whole thing inside a Docker container.

A good practice when building Docker images to be run in the cloud is to use a multibuild setup where the executable is created in a `builder` step and then copied into a different, slimmer image.

This is useful because to build the application, you need Rust, Cargo, and many other software packages. If you were to just keep the executable within this container and run it there, the resulting image would be very large (~1.5 GB in this case).

With a multistage build, however, you can use a very slim base image to actually run the created executable.

First, let’s look at using a slim Debian Buster base image. The following would be a working Dockerfile to do that.


```bash
FROM rust:1.43 as builder

RUN USER=root cargo new --bin rust-docker-web
WORKDIR ./rust-docker-web
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build --release
RUN rm src/*.rs

ADD . ./

RUN rm ./target/release/deps/rust_docker_web*
RUN cargo build --release


FROM debian:buster-slim
ARG APP=/usr/src/app

RUN apt-get update \
    && apt-get install -y ca-certificates tzdata \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8000

ENV TZ=Etc/UTC \
    APP_USER=appuser

RUN groupadd $APP_USER \
    && useradd -g $APP_USER $APP_USER \
    && mkdir -p ${APP}

COPY --from=builder /rust-docker-web/target/release/rust-docker-web ${APP}/rust-docker-web

RUN chown -R $APP_USER:$APP_USER ${APP}

USER $APP_USER
WORKDIR ${APP}

CMD ["./rust-docker-web"]
```

That’s quite a bit of text, so let’s go through it step by step.

As mentioned above, there are two build steps: the `builder`, which simply uses a Rust 1.43 base image, and a second step, which uses Debian.

In the builder step, a simple pseudodependency-caching mechanism is used. First, an empty Rust project is created using `cargo` and then all dependencies (in the form of `Cargo.toml`) are copied into that project.

Then a release-build is triggered before removing everything in the `src` folder. With Docker, for each command inside the `Dockerfile`, a layer is created. On subsequent builds, only layers where the inputs have changed need to be rebuilt.

This is why you build the dependencies independent of your own code — because the dependencies will usually change a lot less often than dependencies. If you only change the code in `src`, none of dependency layer will need to be rebuilt. Since Rust’s compile times are unfortunately still rather long, especially with complex web applications, this can save you quite some time.

A after removing all files in src, copy all files of the project into the Docker working directory, remove the binary built from the dependencies, and trigger another release build — in this case, with the whole code.

Now let’s look at the second part of the `Dockerfile`, where we create a runnable container image.

Most of the commands in this part are cookie-cutter commands you would use for web applications within Docker in any other language.

First, we’ll install some required packages using `apt`, `ca-certificates` and `tzdata`, which is a good baseline for web applications.

Next, expose the application port and create a nonroot user, which is used to run the executable. The most interesting part is the `COPY --from-builder` command, which enables us to actually copy the executable built within the builder step into this container.

At the end, the executable is simply started with the newly created user.

To build this container, simply execute the following.

```bash
docker build -t rust-debian -f ./debian/Dockerfile .
```
You can then execute it using:

```bash
docker run -p 8000:8000 rust-debian
```

This will run the Rust web application, and calling `http://localhost:8000/health` will return OK. Success!

Looking at the output of `docker images`, we can see the following image size:

```bash
rust-debian          latest      d5ae1c61b310     About a minute ago   84.7MB
```
Under 90MB — not too bad. But we can do a lot better. Let’s see how we can do the same with an Alpine base image.

## Docker image (Alpine)

Most of the Alpine-based `Dockerfile` is the same, but there are some important differences.

```bash
FROM ekidd/rust-musl-builder:stable as builder

RUN USER=root cargo new --bin rust-docker-web
WORKDIR ./rust-docker-web
COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml
RUN cargo build --release
RUN rm src/*.rs

ADD . ./

RUN rm ./target/x86_64-unknown-linux-musl/release/deps/rust_docker_web*
RUN cargo build --release


FROM alpine:latest

ARG APP=/usr/src/app

EXPOSE 8000

ENV TZ=Etc/UTC \
    APP_USER=appuser

RUN addgroup -S $APP_USER \
    && adduser -S -g $APP_USER $APP_USER

RUN apk update \
    && apk add --no-cache ca-certificates tzdata \
    && rm -rf /var/cache/apk/*

COPY --from=builder /home/rust/src/rust-docker-web/target/x86_64-unknown-linux-musl/release/rust-docker-web ${APP}/rust-docker-web

RUN chown -R $APP_USER:$APP_USER ${APP}

USER $APP_USER
WORKDIR ${APP}

CMD ["./rust-docker-web"]
```

In the `builder`, the most important change is the base image, where instead of the official Rust image, the [rust-musl-builder](https://hub.docker.com/r/ekidd/rust-musl-builder/) image is used.

The reason for this is that we want to build a static Rust binary that we can just copy over to an Alpine base image. Because Alpine is also built around `musl-libc`, an executable using the official Docker image won’t work.

Besides that, the builder stage is very similar. The only thing that changes is the path to the executable.

Within the runner stage, we now use `alpine:latest` as a base image and have to use `apk` and some different commands to create the user to run the application. The steps themselves are basically the same, but the commands and paths are a bit different.

The basic structure of this `Dockerfile` is very similar to the Debian-based one, so building and running it also works the same way.

```bash
docker build -t rust-alpine -f ./alpine/Dockerfile .
docker run -p 8000:8000 rust-alpine
```

Again, upon opening `http://localhost:8000/health`, you’ll get a successful response.

The reason we switched to Alpine in the first place was the image size, so let’s see where we land here.

```bash
rust-alpine          latest      492d0305ebcb     About a minute ago   16MB
```

16MB — that’s an 80 percent decrease in image size. However, while this decrease in size is quite nice, what are the tradeoffs involved here? Are there any disadvantages to using the Alpine image versus the Debian one?

Let’s have a look at the implications involved when choosing which image to use as a base.

## Choosing a base image

The clear benefit of Alpine is the lower image size. However, as described in the [rust-musl-builder](https://hub.docker.com/r/ekidd/rust-musl-builder/) docs, there have been issues in the past with making OpenSSL work. While that has been fixed, it’s possible that you’ll run into trouble if you depend on C libraries, which will be easier to handle with the standard toolchain.

There are also severe performance issues related to memory allocation using the Alpine-based image. Andy Grove tracked this back to the use of musl. In some cases, it helped to switch to the jemalloc allocator.

I personally didn’t run into the aforementioned issues, but I did notice a lower memory footprint when running the app inside the Alpine container compared to Debian. These metrics will be different depending on the workload you want to run, so it’s impossible to give silver-bullet advice here.

In general, the decision of which stack to use is yours and yours alone. It needs to be based on the tradeoffs that are important to your use case.

As with any other ops decision, there is no one-size-fits all for production web services. You need to test, benchmark, and profile for your use case and then decide what is most suitable.

## Conclusion

In this guide, we walked through how to containerize a Rust web application in two different ways that have distinctly different trade-offs.

The two approaches outlined above are not the only ways you can go about this and, as mentioned before, the approach you take for a production service will have to be based on your own unique needs and data. However, the examples in this post are a good starting point to get your feet wet using Docker with Rust.

You can find the full code for this example on [GitHub](https://github.com/zupzup/rust-docker-web).

