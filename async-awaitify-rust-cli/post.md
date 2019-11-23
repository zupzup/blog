In a [previous post](https://zupzup.org/rust-gitlab-mr-creator/) we built a CLI app using rust. This app used both an HTTP API using the asynchronous [hyper](https://github.com/hyperium/hyper) library, as well as GIT using the synchronous [git2](https://github.com/rust-lang/git2-rs).

In this post, we'll port the whole app to the new `async/await` syntax in the hope that the interplay between asynchronous and synchronous program flow becomes easier to handle as well as that the complexity of the whole app is reduced somewhat with increased readability as a side-benefit.

Please keep in mind that at this point `async/await` has just very recently been stabilized and the whole ecosystem (tokio, hyper..) are still in alpha stages towards supporting it. That means that some of the nuances of this example could change over time towards full stabilization across the ecosystem.

Anyway, let's get started!

## Moving to Async/Await 

We'll start by updating dependencies and then we'll move on to change the `hyper` parts to `async/await`. Afterwards, we'll see how we can simplify the whole application, hopefully reducing some of the `.clone()` calls and making it more readable overall.

The data model and `clap` setup for the CLI won't change at all, so it'll be omitted in this post.

The first part we'll touch is the `http` module, which houses our API requests to the Gitlab API. We should be able to transform this module without touching the other parts of the application.

Before we move over the first module, we need to update dependencies. As mentioned above, right now the whole ecosystem is basically in `alpha` mode before async/await is stabilized, so we use the latest alphas at this point:

```bash
hyper = { version = "0.13.0-a.4", features = ["unstable-stream"]}
hyper-tls = { version = "0.4.0-a.4" }
tokio = { version = "=0.2.0-alpha.6" }
futures-preview = { version = "0.3.0-alpha.19", features = ["async-await"] }
futures-core-preview = "=0.3.0-alpha.19"
futures-io-preview = "=0.3.0-alpha.19"
futures-util-preview = "=0.3.0-alpha.19"
```

We use the `unstable-stream` feature of hyper to be able to use `try_concat().await` on a `hyper::Body`, which waits for the whole chunk of data at once (asynchronously).

The first function we'll tackle is `fetch_projects`, but in order to do so, we'll define some convenience types first:

```rust
type Result<T> = std::result::Result<T, HttpError>;

#[derive(Debug)]
pub enum HttpError {
...
}
```

We'll use the `HttpError` error type and re-define `Result` to be an `std::result::Result` with that error and `T`, this will reduce the amount of types we'll have to write. The actual function is modified to this:

```rust
pub async fn fetch_projects(
    config: &Config,
    access_token: &str,
    domain: &str,
) -> Result<Vec<ProjectResponse>> {
    let projects_raw = fetch(config, access_token, domain, 20).await?;
    let mut result: Vec<ProjectResponse> = Vec::new();
    for p in projects_raw {
        let mut data: Vec<ProjectResponse> = serde_json::from_slice(&p)?;
        result.append(&mut data);
    }
    Ok(result)
}
```

We fetch the projects using the `fetch` function, which we'll look at next and then just deserialize the projects to a Vector of ProjectResponse. Notice how clean it looks compared to the previous version:

```rust
pub fn fetch_projects(
    config: Config,
    access_token: String,
    domain: String,
) -> impl Future<Item = Vec<ProjectResponse>, Error = Error> {
    fetch(config, access_token, domain, 20)
        .and_then(|bodies| {
            let mut result: Vec<ProjectResponse> = Vec::new();
            future::join_all(bodies)
                .and_then(|bods| {
                    for b in bods {
                        let bytes = b.into_bytes();
                        let mut data: Vec<ProjectResponse> = serde_json::from_slice(&bytes)
                            .expect("can't parse data to project response");
                        result.append(&mut data);
                    }
                    future::ok(result)
                })
                .map_err(|e| format_err!("req err: {}", e))
        })
        .map_err(|err| err)
}
```

All the nesting is gone and it reads very much like synchronous code. Granted, in the previous version we had a list of body futures to execute before deserializing, but even without that, the new async/awaitified version looks much cleaner.

Let's look at `fetch` next.

In `fetch`, we create a `hyper` https client, build the URL we need to fetch based on the domain, make the first request there and, if there are more pages (in Gitlab, all resources are paged by default), then we fetch all pages until the end.

```rust
async fn fetch(
    config: &Config,
    access_token: &str,
    domain: &str,
    per_page: i32,
) -> Result<Vec<Bytes>> {
    let https = HttpsConnector::new()?;
    let client = Client::builder().build::<_, hyper::Body>(https);
    let group = config.group.as_ref();
    let user = config.user.as_ref();
    let uri = match group {
        Some(v) => format!(
            "https://gitlab.com/api/v4/groups/{}/{}?per_page={}",
            v, domain, per_page
        ),
        None => match user {
            Some(u) => format!(
                "https://gitlab.com/api/v4/users/{}/{}?per_page={}",
                u, domain, per_page
            ),
            None => "invalid url".to_string(),
        },
    };
    let req = Request::builder()
        .uri(uri)
        .header("PRIVATE-TOKEN", access_token.to_owned())
        .body(Body::empty())?;
    let res = client.request(req).await?;
    if !res.status().is_success() {
        return Err(HttpError::UnsuccessFulError(res.status()));
    }
    let pages: &str = match res.headers().get("x-total-pages") {
        Some(v) => match v.to_str() {
            Ok(v) => v,
            _ => "0",
        },
        None => "0",
    };
    let p = match pages.parse::<i32>() {
        Ok(v) => v,
        Err(_) => 0,
    };
    let mut result: Vec<Bytes> = Vec::new();
    let body = res.into_body().try_concat().await?;
    result.push(body.into_bytes());
    let mut futrs = Vec::new();
    for page in 2..=p {
        futrs.push(fetch_paged(&config, &access_token, &domain, &client, page));
    }
    let paged_results = future::join_all(futrs).await;
    for r in paged_results {
        let str = match r {
            Ok(v) => v,
            Err(_) => return Err(HttpError::UnsuccessFulError(hyper::StatusCode::INTERNAL_SERVER_ERROR)),
        };
        result.push(str);
    }
    return Ok(result);
}
```

The first part of the function didn't change much, besides the function being `async` now - we still get values from the config and create the hyper client. The interesting part comes when we execute the actual HTTP request.

We can simply use `client.request(req).await?` to get the response. Then, as before, we parse the `pages` in order to know how many paged requests we'll have to make. Another nice thing is the use of `try_concat().await` from the `TryStreamExt` within the `futures` crate, which lets us wait for the complete body stream to finish. This way we don't have to manually concatenate the byte chunks.

Next up we create a vector of futures and put the `fetch_paged` futures in there. We'll go over `fetch_paged` next, but it basically just does paged requests to the same resource. We execute all these futures using `join_all`, which gives us a vector of results, which we push into our result vector to return.

One change we made here, is that we don't return `Body` objects anymore, but response bytes, which is a nicer API, as the only thing we plan to do with the responses is json deserialization anyway.

Ok, next up, we'll look at `fetch_paged`. The changes made in there are quite similar to `fetch`. We await the response and use `try_concat()` to get our response as bytes, returning that:

```rust
async fn fetch_paged(
    config: &Config,
    access_token: &str,
    domain: &str,
    client: &hyper::Client<HttpsConnector<HttpConnector>>,
    page: i32,
) -> Result<Bytes> {
    let group = match config.group.as_ref() {
        Some(v) => v,
        None => return Err(HttpError::ConfigError())
    };
    let req = Request::builder()
        .uri(format!(
            "https://gitlab.com/api/v4/groups/{}/{}?per_page=20&page={}",
            group, domain, page
        ))
        .header("PRIVATE-TOKEN", access_token)
        .body(Body::empty())?;
    let res = client.request(req).await?;
    if !res.status().is_success() {
        return Err(HttpError::UnsuccessFulError(res.status()));
    }
    let body = res.into_body().try_concat().await?;
    return Ok(body.into_bytes());
}
```

The last function in the `http` module is `create_mr`, which we use to finally create the merge request after we pushed to GitLab.

This is how it looks after the change:

```rust
pub async fn create_mr(payload: &MRRequest<'_>, config: &Config) -> Result<String> {
    let https = HttpsConnector::new()?;
    let client = Client::builder().build::<_, hyper::Body>(https);
    let uri = format!(
        "https://gitlab.com/api/v4/projects/{}/merge_requests",
        payload.project.id
    );
    let labels = config
        .mr_labels
        .as_ref()
        .unwrap_or(&Vec::new())
        .iter()
        .fold(String::new(), |acc, l| format!("{}, {}", acc, l));

    let mr_payload = MRPayload {
        id: &format!("{}", payload.project.id),
        title: &payload.title,
        description: &payload.description,
        target_branch: &payload.target_branch,
        source_branch: &payload.source_branch,
        labels: &labels,
        squash: true,
        remove_source_branch: true,
    };
    let json = serde_json::to_string(&mr_payload)?;
    let req = Request::builder()
        .uri(uri)
        .header("PRIVATE-TOKEN", payload.access_token.to_owned())
        .header("Content-Type", "application/json")
        .method(Method::POST)
        .body(Body::from(json))?;
    let res = client.request(req).await?;
    if !res.status().is_success() {
        return Err(HttpError::UnsuccessFulError(res.status()));
    }
    let body = res.into_body().try_concat().await?;
    let bytes = body.into_bytes();
    let data: MRResponse = serde_json::from_slice(&bytes)?;
    Ok(data.web_url)
}
```

Besides the `async/await` changes, which are the same as in the functions before, the biggest change here is, that we could get rid of a lot of `.clone()` calls while creating the payload, which gets passed around.

Another thing to mention from this small rewrite is, that we added some nicer errors. Before, we used the `failure` crate to return custom error messages. In this case, we introduce a custom `HttpError` enum type and handle errors in a clearer way:


```rust
type Result<T> = std::result::Result<T, HttpError>;

#[derive(Debug)]
pub enum HttpError {
    UnsuccessFulError(hyper::StatusCode),
    ConfigError(),
    HyperError(hyper::Error),
    HyperTLSError(hyper_tls::Error),
    HyperHttpError(hyper::http::Error),
    JsonError(serde_json::Error),
}

impl Error for HttpError {
    fn description(&self) -> &str {
        match *self {
            HttpError::UnsuccessFulError(..) => "unsuccessful request",
            HttpError::ConfigError(..) => "invalid config provided - no group",
            HttpError::HyperError(..) => "hyper error",
            HttpError::HyperTLSError(..) => "hyper tls error",
            HttpError::HyperHttpError(..) => "hyper http error",
            HttpError::JsonError(..) => "serde json error",
        }
    }
    fn cause(&self) -> Option<&dyn Error> {
        match *self {
            HttpError::UnsuccessFulError(..) => None,
            HttpError::ConfigError(..) => None,
            HttpError::HyperError(ref e) => Some(e),
            HttpError::HyperTLSError(ref e) => Some(e),
            HttpError::HyperHttpError(ref e) => Some(e),
            HttpError::JsonError(ref e) => Some(e),
        }
    }
}

impl fmt::Display for HttpError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}: ", self.description())?;
        match *self {
            HttpError::UnsuccessFulError(ref v) => write!(f, "unsuccessful request: {}", v),
                HttpError::ConfigError(..) => write!(f, "invalid config found - no group"),
                HttpError::HyperError(ref e) => write!(f, "{}", e),
                HttpError::HyperTLSError(ref e) => write!(f, "{}", e),
                HttpError::HyperHttpError(ref e) => write!(f, "{}", e),
                HttpError::JsonError(ref e) => write!(f, "{}", e),
        }
    }
}
```

Of course, these trait implementations could be simply derived using a crate like [thiserror](https://github.com/dtolnay/thiserror), but it has some educational value to write them on your own as well at least once.

Alright, that about covers the `http` module. Changing it to async/await wasn't very hard and made the code a lot more readable and ergonomic to work with in my opinion.

Next, we'll look at the `main.rs` file and what we changed there to async-awaitify the app.

The part of the code dealing with the command line arguments using `clap` and the `toml` configuration stayed exactly the same, so I won't mention it here again. We could have used an asynchronous `fs::read` for reading the configuration file of course, but we'll stick to async networking in this post.

Let's start with the implementation of the `create_mr` function in `main.rs`:

```rust
fn create_mr(
    config: &Config,
    actual_remote: &str,
    access_token: &str,
    title: &str,
    description: &str,
    target_branch: &str,
    current_branch: &str,
) {
    let rt = Runtime::new().expect("tokio runtime can be initialized");
    rt.block_on(async move {
        let projects = match http::fetch_projects(&config, &access_token, "projects").await {
            Ok(v) => v,
            Err(e) => return println!("could not fetch projects, reason: {}", e)
        };
        let mut actual_project: Option<&ProjectResponse> = None;
        for p in &projects {
            if p.ssh_url_to_repo == actual_remote {
                actual_project = Some(p);
                break;
            }
            if p.http_url_to_repo == actual_remote {
                actual_project = Some(p);
                break;
            }
        }
        let project = actual_project.expect("couldn't find this project on gitlab");
        let mr_req = MRRequest {
            access_token,
            project,
            title,
            description,
            source_branch: current_branch,
            target_branch,
        };
        match http::create_mr(&mr_req, &config).await {
            Ok(v) => println!("Pushed and Created MR Successfully - URL: {}", v),
            Err(e) => println!("Could not create MR, Error: {}", e)
        };
    });
}
```

So, I said in the beginning, that we're going to async/awaitify the code - however, due to the nature of the application, this step actually needs to block. So while we execute the HTTP call asynchronously, we still have to wait for the result, as it's the last thing that happens in the program flow (creating an actual merge request).

Because we create this merge request in the callback of the `git2` push functionality, and because `git2` doesn't have an asynchronous interface for now, we need to block manually here. We do this by using tokio's `rt.block_on`, which takes a future and blocks until it's completed.

We can create this future inline using an `async move` block, capturing all parameters, which is convenient. Inside, we fetch the projects, prepare the payload and call our `http` module's `create_mr` function, logging on success and error.

The other changes in this project aren't directly related to async/await. It's mostly about error handling, using the `Result<T> = std::result::Result<T, AppError>` type as result and mapping errors to our custom Error type:

```rust
#[derive(Debug)]
pub enum AppError {
    AccessTokenNotFoundError(),
    IOError(std::io::Error),
    TomlParseError(toml::de::Error),
    GitError(String),
    HttpError(http::HttpError),
}
```

The flow in the `main.rs` function stays exactly the same - the biggest differences are, that we use fewer clones due to defining the MR payload using lifetimes.

So that's it - the full code can be found [here](https://github.com/zupzup/gitlab-push-and-mr/tree/async_await)


## Conclusion

I was surprised by how seamless migrating the existing futures-based implementation to async/await was. I guess this was, because the application didn't use anything fancy in terms of futures. But still it's nice to see that it's not too much of a hassle to move over to the new API. :)

Seeing the new async/await API come to live and the community-wide adoption in the ecosystem is exciting. I believe this will lead to many new cool projects, especially in the web-space, which is great  both for the Rust community as well as for the Web-development community.


#### Resources

* [code](https://github.com/zupzup/gitlab-push-and-mr/tree/async_await)
* [hyper](https://github.com/hyperium/hyper)
* [git2](https://github.com/rust-lang/git2-rs)
* [thiserror](https://github.com/dtolnay/thiserror)
