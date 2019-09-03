In this post I'll describe the first CLI tool I built with `rust`. The program in question pushes the current `git` branch and creates an MR for it on `Gitlab`

In terms of libraries, we'll use [clap](https://github.com/clap-rs/clap) for parsing command line arguments, [hyper](https://github.com/hyperium/hyper) for making HTTP requests to the gitlab API and [git2](https://github.com/rust-lang/git2-rs) for interacting with the underlying git repository.

I decided to use `hyper` in this case, instead of a higher-level library such as `reqwest`, because I wanted to get to know it a little better.

An interesting aspect of this project is, that it uses both asynchronous (http) and synchronous (git2) execution flow, which will make it a nice fit for moving to `async/await` in a further post.

Let's get started!

## Implementation 

First, we'll start by setting up the command line application and the different arguments it can take. For this, we use `clap`, which is a fantastic library for exactly this purpose.

```rust
let matches = App::new("Gitlab Push-and-MR")
    .version("1.0")
    .arg(
        Arg::with_name("description")
            .short("d")
            .long("description")
            .value_name("DESCRIPTION")
            .help("The Merge-Request description")
            .takes_value(true),
    )
    .arg(
        Arg::with_name("title")
            .short("t")
            .required(true)
            .long("title")
            .value_name("TITLE")
            .help("The Merge-Request title")
            .takes_value(true),
    )
    .arg(
        Arg::with_name("target_branch")
            .short("b")
            .long("target-branch")
            .value_name("TARGETBRANCH")
            .help("The Merge-Request target branch")
            .takes_value(true),
    )
    .get_matches();
let title = matches
    .value_of("title")
    .expect("title needs to be provided");
let description = matches.value_of("description").unwrap_or("");
let target_branch = matches.value_of("target_branch").unwrap_or("master");
```

There are three options:

- `description` - The MR description to set
- `title` - The MR title to set
- `target branch` - The branch to merge the MR to, defaults to `master`

Also, in order to push to git and create an MR on gitlab, we'll need credentials. For git, we'll use `ssh` authentication and for gitlab we need to provide an access token to the application.

In a fancy version of this, we could use the OS's key management to handle the `ssh` password for us, but to keep things simple, we provide it via an environment variable for now.

Anyway, we need more configuration in the form of files and environment variables:

* `access_token` - first line read from a `./secret` file
* `config` - toml file read from `./config.toml`
* `ssh-key` - the location of the SSH key from the `SSH_KEY_FILE` env variable
* `ssh-password` - the password for the ssh key from the `SSH_PASS` env variable

Now, in production-grade software, secrets such as these should be handled in a more consistent and secure way, or, optimally handled using external tools (e.g. password managers etc.). But as this is simply a learning project, we'll try to use several different ways to load configurations to see how they work.

Getting the access token from the `.secrets` file is rather simple:

```rust
fn get_access_token() -> Result<String, Error> {
    let file = File::open(SECRETS_FILE).expect("Could not read access token file");
    let buf = BufReader::new(file);
    let lines: Vec<String> = buf
        .lines()
        .take(1)
        .map(std::result::Result::unwrap_or_default)
        .collect();
    if lines[0].is_empty() {
        return Err(format_err!("access token mustn't be empty"));
    }
    Ok(lines[0].to_string())
}
```

We simply open the file and try to read the first line, returning it.

Loading the `config.toml` is even easier:

```rust
fn get_config() -> Result<Config, Error> {
    let data = fs::read_to_string(CONFIG_FILE)?;
    let config: Config = toml::from_str(&data)?;
    Ok(config)
}
```

Again, we read the file and then parse it to our configuration structure, which looks like this:

```rust
#[derive(Debug, Deserialize, Clone)]
pub struct Config {
    pub group: Option<String>,
    pub user: Option<String>,
    pub mr_labels: Option<Vec<String>>,
}
```

The two methods are called from main as follows:

```rust
let access_token = get_access_token().expect("could not get access token");
let config = get_config().expect("could not read config");
```

Alright, with our config all set up, let's get to the `git` part of the application. First, we'll check if the current folder is even a git repository:

```rust
let repo = Repository::open("./").expect("current folder is not a git repository");
```

Then, we'll try to detect the current branch, because we'll need it when creating the MR later as the source branch:

```rust
let current_branch = get_current_branch(&repo).expect("could not get current branch");
```

With the `get_current_branch`-function:

```rust
fn get_current_branch(repo: &Repository) -> Result<String, Error> {
    let branches = repo.branches(None).expect("can't list branches");
    branches.fold(
        Err(format_err!("couldn't find current branch")),
        |acc, branch| {
            let b = branch?;
            if b.0.is_head() {
                let name = b.0.name()?;
                return match name {
                    Some(n) => Ok(n.to_string()),
                    None => return acc,
                };
            }
            acc
        },
    )
}
```

Here, we first list all branches and then search them for the branch which has `is_head` set, extracting its name, handling potential errors on the way there.

Alright, the next thing to do is to push the current changeset. We'll default to the `origin` remote in this case for simplicity, but this could also be added as a configuration option.

A short disclaimer here - this likely will only work for very standard git repositories. The `git2` library and git in general provide a staggering amount of things to configure and do in different ways, but going
down that rabbit-hole would far exceed the scope of this post, so we'll stick to pushing to `origin` only.

```rust
let mut remote = repo
    .find_remote("origin")
    .expect("origin remote could not be found");
let actual_remote = String::from(remote.url().expect("could not get remote URL"));
```

We simply select the `origin` remote and store it's url. We'll use this URL later on to identify the current gitlab project from a list of fetched gitlab projects.

In order to `push` to the gitlab repository, we need credentials and as mentioned above, we'll use ssh authentication. In order to register these credentials, we need push-options in the form of callback functions:


```rust
let mut push_opts = PushOptions::new();
let mut callbacks = RemoteCallbacks::new();
callbacks.credentials(git_credentials_callback);
push_opts.remote_callbacks(callbacks);
```

With the `git_credentials_callback` looking like this:

```rust
fn git_credentials_callback(
    _user: &str,
    user_from_url: Option<&str>,
    cred: git2::CredentialType,
) -> Result<git2::Cred, git2::Error> {
    let user = user_from_url.unwrap_or("git");
    if cred.contains(git2::CredentialType::USERNAME) {
        return git2::Cred::username(user);
    }
    let key_file = env::var("SSH_KEY_FILE").expect("no ssh key file provided");
    let passphrase = env::var("SSH_PASS").expect("no ssh pass provided");
    git2::Cred::ssh_key(
        user,
        None,
        std::path::Path::new(&key_file),
        Some(&passphrase),
    )
}
```

Here we get the path to the ssh key file and the ssh password from env variables and execute the `git2` ssh credential mechanism.

With authentication out of the way, we simply push the current branch with our above created push options:

```rust
remote
.push(
    &[&format!("refs/heads/{}", current_branch.to_string())],
    Some(&mut push_opts),
 )
.expect("could not push to origin");
```

Alright, in order to, after the successful push, create the merge request, we need another callback function:

```rust
callbacks.push_update_reference(move |refname, _| {
    create_mr(
        config.clone(),
        remote_clone.clone(),
        access_token.clone(),
        title.to_owned(),
        description.to_owned(),
        target_branch.to_owned(),
        branch_clone.clone(),
    );
    Ok(())
});
```

So basically, this callback is evoked after the pull has successfully happened, and we simply call the `create_mr` function:

```rust
fn create_mr(
    config: Config,
    actual_remote: String,
    access_token: String,
    title: String,
    description: String,
    target_branch: String,
    current_branch: String,
) {
    let fut = http::fetch_projects(
        config.clone(),
        access_token.clone(),
        "projects".to_string(),
    );
	....
}
```

In this function, we first fetch all projects for the configured gitlab user and parse them to `ProjectResponse`s:

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct ProjectResponse {
    pub id: Number,
    pub name: String,
    pub ssh_url_to_repo: String,
    pub http_url_to_repo: String,
}
```

We do this in order to find the current git repositories' gitlab project counterpart, because we need the project ID for creating the MR.

The code for interacting with the gitlab API via HTTP is in the `http` module. There we have the `fetchProjects` function:


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

However, fetching projects isn't quite that simple, as in gitlab's API, all fetching operations are paged by default. That means, that in order to be sure we fetch all available entity objects, we need to fetch in a paged manner.

For this purpose, we have the `fetch` function, which calls the `fetch_paged` function:

```rust
fn fetch(
    config: Config,
    access_token: String,
    domain: String,
    per_page: i32,
) -> impl Future<Item = Vec<Concat2<Body>>, Error = Error> {
    let https = HttpsConnector::new(4).expect("https connector works");
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
        .body(Body::empty())
        .expect("request creation works");
    client
        .request(req)
        .map_err(|e| format_err!("req err: {}", e))
        .and_then(move |res: Response<Body>| {
            if !res.status().is_success() {
                return future::err(format_err!("unsuccessful fetch request: {}", res.status()));
            }
            future::ok(res)
        })
        .and_then(move |res: Response<Body>| {
            let mut result: Vec<Concat2<Body>> = Vec::new();
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
            let body: Body = res.into_body();
            result.push(body.concat2());
            let mut futrs = Vec::new();
            for page in 2..=p {
                futrs.push(fetch_paged(&config, &access_token, &domain, &client, page));
            }
            future::join_all(futrs)
                .and_then(move |bodies| {
                    for b in bodies {
                        result.push(b);
                    }
                    future::ok(result)
                })
                .map_err(|e| format_err!("requests error: {}", e))
        })
}
```

Ok - this is a bit more complex, so let's go over it step-by-step.

The first step is to setup the `hyper` client using the `HTTPSConnector` - so far, so good. Then, we build the URL of the resource to be fetched (like projects). This URL depends on whether we are using a group or a user as the basis in our configuration, so we match on that.

Then, we create the actual HTTP request using the configured `access_token` in the header. If we get a successful response, we parse out the response's `x-total-pages` header, which will tell us how many pages there are to fetch.

For each of these pages, we create a future using the `fetch_paged` function, which we'll look at next. Then we add all the futures to a vector and use `future::join_all` to execute all of them. If there is an error anywhere, we stop and propagate the error up the chain, otherwise we collect the bodies of all responses in a `result` vector.

The `fetch_paged` function basically just executes the above request with a given page and returns the response body future:

```rust
fn fetch_paged(
    config: &Config,
    access_token: &str,
    domain: &str,
    client: &hyper::Client<HttpsConnector<HttpConnector>>,
    page: i32,
) -> impl Future<Item = Concat2<Body>, Error = Error> {
    let group = config.group.as_ref().expect("group is configured");
    let req = Request::builder()
        .uri(format!(
            "https://gitlab.com/api/v4/groups/{}/{}?per_page=20&page={}",
            group, domain, page
        ))
        .header("PRIVATE-TOKEN", access_token)
        .body(Body::empty())
        .expect("request can be created");
    client
        .request(req)
        .map_err(|e| format_err!("req err: {}", e))
        .and_then(|res| {
            if !res.status().is_success() {
                return future::err(format_err!(
                    "unsuccessful fetch paged request: {}",
                    res.status()
                ));
            }
            let body = res.into_body().concat2();
            future::ok(body)
        })
}
```

Ok, cool! So now we should have a list of all the gitlab projects for the user / in the group we configured. That means we can make progress on understanding our `create_mr` function from above:

```rust
fn create_mr(
    config: Config,
    actual_remote: String,
    access_token: String,
    title: String,
    description: String,
    target_branch: String,
    current_branch: String,
) {
    let fut = http::fetch_projects(config.clone(), access_token.clone(), "projects".to_string())
        .and_then(move |projects: Vec<ProjectResponse>| {
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
            http::create_mr(&mr_req, &config)
        })
        .map_err(|e| {
            println!("Could not create MR, Error: {}", e);
        })
        .and_then(|url: String| {
            println!("Pushed and Created MR Successfully - URL: {}", url);
            future::ok(())
        });
    rt::run(fut);
}
```

We iterate the list of projects and try to match the ssh and http url of the projects to our actual remote URL collected before. If we can't find the current project on gitlab, we exit here. Otherwise, the next step is to create the Merge Request.

This is done by creating an `MRRequest` object with all the data we'll need to create the MR:

```rust
#[derive(Debug, Serialize, Clone)]
pub struct MRRequest<'a> {
    pub access_token: String,
    pub project: &'a ProjectResponse,
    pub title: String,
    pub description: String,
    pub source_branch: String,
    pub target_branch: String,
}
```

This data object is then passed to the `create_mr` function in the http module:

```rust
pub fn create_mr(
    payload: &MRRequest,
    config: &Config,
) -> impl Future<Item = String, Error = Error> {
    let https = HttpsConnector::new(4).expect("https connector works");
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
        id: format!("{}", payload.project.id),
        title: payload.title.clone(),
        description: payload.description.clone(),
        target_branch: payload.target_branch.clone(),
        source_branch: payload.source_branch.clone(),
        labels,
        squash: true,
        remove_source_branch: true,
    };
    let json = serde_json::to_string(&mr_payload).expect("payload can be stringified");
    let req = Request::builder()
        .uri(uri)
        .header("PRIVATE-TOKEN", payload.access_token.to_owned())
        .header("Content-Type", "application/json")
        .method(Method::POST)
        .body(Body::from(json))
        .expect("request can be created");
    client
        .request(req)
        .map_err(|e| format_err!("req err: {}", e))
        .and_then(move |res: Response<Body>| {
            if !res.status().is_success() {
                return future::err(format_err!(
                    "unsuccessful create mr request: {}",
                    res.status()
                ));
            }
            let body = res.into_body();
            future::ok(body)
        })
        .and_then(|body: Body| {
            body.concat2()
                .map_err(|e| format_err!("requests error: {}:", e))
        })
        .and_then(|body| {
            let bytes = body.into_bytes();
            let data: MRResponse =
                serde_json::from_slice(&bytes).expect("can't parse data to merge request response");
            future::ok(data.web_url)
        })
        .map_err(|e| format_err!("requests error: {}", e))
}
```

Ok, this is again a bit longer and more involved, so let's look at what's happening here.

First, we build the gitlab API url using our previously determined project id and grab the configured labels from the configuration.

After that, we create an `MRPayload` object, which represents the JSON payload we'll send to gitlab to create the MR. It includes the project id, title, description, target and source branches, the labels and some settings.

This payload object is then serialized to JSON. With that JSON, we do the actual request and simply propagate if it was successful, or not.

Very nice - if pushing and creating the MR worked, we're done. :)

The full example code can be found [here](https://github.com/zupzup/gitlab-push-and-mr)

## Conclusion 

I had a lot of fun building this small utility in rust. This was the first time writing a CLI in rust, which wasn't a big issue at all due to `clap` being awesome, but also the first time I 
used hyper and futures without a framework doing all the work for me, which was a bit more challenging. ;)

I'm sure this solution is far from optimal in regards to clones, structure and error-handling, but it works and to my (rather inexperienced) rust-eyes, it could certainly look worse.

In a future post I plan to rewrite this using `async/await`, which should be a nice use-case, as we use both sync (git2) and async (hyper) flow in the application and I expect
there will be less issues threading values through the future chains with the new API - I'm excited to test it out either way. :)


#### Resources

* [Code Example](https://github.com/zupzup/gitlab-push-and-mr)
* [hyper](https://github.com/hyperium/hyper)
* [clap](https://github.com/clap-rs/clap)
* [gitlab](https://gitlab.com/)
* [git2](https://github.com/rust-lang/git2-rs)
