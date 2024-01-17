*This post was originally posted on the [LogRocket](https://blog.logrocket.com/configuration-management-in-rust-web-services/) blog on 22.09.2020 and was cross-posted here by the author.*

For any nontrivial web service, the ability to manipulate configuration values is essential. This can range from setting the host and credentials of a database over the cron expression of a job to, quite simply, defining the actual port the application should be started on. All these values could be subject to change, and it would be highly inefficient to go through the processes of a code change and deployment every time.

For this reason, it’s useful to have external configuration files, which are loaded at startup time and set these values for the current run. This way, you only have to change the value in the configuration file and restart the service. If you have a fancy reload mechanism for your configuration, you might not even have to restart.

In this tutorial, we’ll walk through the basics of configuration management in a Rust web application. We’ll build an example application using [warp](https://github.com/seanmonstar/warp) to create a web server, but the concepts will apply identically to any other method of spinning up a web app.

The demo will show off the following features.

* Hierarchical configuration
* Environment variable overrides
* Nested data structures
* Multiple file formats
* Manual overrides
* Global and local access to the config

And we won’t even have to break a sweat, since the fantastic [config-rs](https://github.com/mehcode/config-rs) crate already provides these features and more out of the box, with just a bit of configuration necessary.

Let’s get started!

## Setup

To follow along, all you need is a recent Rust installation (1.39+) and a tool to send HTTP requests, such as cURL.

First, create a new Rust project:

```bash
cargo new rust-config-example
cd rust-config-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need.

```toml
[dependencies]
tokio = { version = "0.2", features = ["macros", "rt-threaded"] }
warp = "0.2"
config = "0.10"
serde = {version = "1.0", features = ["derive"] }
lazy_static = "1.4"
```

The web service itself will be written in warp, which uses tokio internally. To manage our configuration, we will use the fantastic [config-rs](https://github.com/mehcode/config-rs) crate. The `lazy_static` crate will enable us to lazily initialize the config as a shared global, which is one of the two ways we’ll demonstrate to propagate the configuration through the application.

## Basic configuration setup

Let’s start by defining the values we would like to configure. The `config` crate supports hierarchical overrides, meaning we can define a default configuration and then, if a different file is used depending on the environment, override some values.

Below is the example default configuration in `./config/Default.toml`.

```toml
[server]
port = "8080"
url = "http://localhost:8080"

[log]
level = "info"

[[rules]]
    name = "all"
    rule_set = [
        ">5",
        "<100",
    ]

[[rules]]
    name = "none"
    rule_set = []
```

Nothing too fancy here. We defined two properties on the `server` key, the log level, and, for whatever reason, a set of rules.

If we wanted to override, say, the log level for our local development runs, we could create a file, `./config/Development.toml`, that contains the following.

```toml
[log]
level = "debug"
```

Loading this `Development` configuration would set all values exactly as in `Default`, except for `log.level`, which would be set to debug instead of `info`.

This is useful because it enables you to define some basic properties that won’t change once in the default configuration and override the ones that change in specialized configs.

For example, `Testing` and `Production` configs would look as follows in this case.

```toml
[server]
url = "http://testing.example.org"
```

```toml
[server]
url = "http://example.org"
```

For each environment, there must be a file in `./config`.

## Loading the configuration

With these configuration files created, let’s take a look at how to load this into a Rust application.

Create a `settings.rs` file in the new Rust project. Then, create data structures for the config values outlined above.

```rust
use config::{Config, ConfigError, Environment, File};

#[derive(Debug, Deserialize, Clone)]
pub struct Log {
    pub level: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct Server {
    pub port: u16,
    pub url: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct Rule {
    pub name: String,
    pub rule_set: Vec<String>,
}

#[derive(Debug, Deserialize, Clone)]
pub struct Settings {
    pub server: Server,
    pub rules: Vec<Rule>,
    pub log: Log,
    pub env: ENV,
}

const CONFIG_FILE_PATH: &str = "./config/Default.toml";
const CONFIG_FILE_PREFIX: &str = "./config/";
```

These structs mirror the fields in `Default.toml`. As you can see, we’re can use different data types and even nested structures.

We also defined two constants for loading the configuration: the default configuration file and the folder name.

But how do we get from these config files to filled-out structs? We’ll get to that shortly.

For now, let’s define the `ENV` data type present in the `Settings` struct.

```rust
#[derive(Clone, Debug, Deserialize)]
pub enum ENV {
    Development,
    Testing,
    Production,
}

impl fmt::Display for ENV {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ENV::Development => write!(f, "Development"),
            ENV::Testing => write!(f, "Testing"),
            ENV::Production => write!(f, "Production"),
        }
    }
}

impl From<&str> for ENV {
    fn from(env: &str) -> Self {
        match env {
            "Testing" => ENV::Testing,
            "Production" => ENV::Production,
            _ => ENV::Development,
        }
    }
}
```

This is entirely optional, but I find it quite useful. It’s just an enum mapping the environment strings. Adding this to the configuration is nice if you have some logic you might want to enable or disable based on the environment you’re running. You could also conceivably have a huge enum with feature flags you might want to switch on or off. Switches like these should be used sparingly, but they can be quite handy.

Now let’s load the config.

```rust
impl Settings {
    pub fn new() -> Result<Self, ConfigError> {
        let env = std::env::var("RUN_ENV").unwrap_or_else(|_| "Development".into());
        let mut s = Config::new();
        s.set("env", env.clone())?;

        s.merge(File::with_name(CONFIG_FILE_PATH))?;
        s.merge(File::with_name(&format!("{}{}", CONFIG_FILE_PREFIX, env)))?;

        // This makes it so "EA_SERVER__PORT overrides server.port
        s.merge(Environment::with_prefix("ea").separator("__"))?;

        s.try_into()
    }
}
```

First, we checked whether a `RUN_ENV` environment variable was set (if not, we’d default to `Development`). This is convenient because we’ll frequently run the app locally, but it also means that if you don’t specify an environment in production, it will start with the `Development` config, which might not be what you want. In our case, it might be more sensible to simply fail here and throw an error explaining that this env variable needs to be set to `Development`, `Testing`, or `Production`.

Next, we created a default `config::Config` and manually set the `ENV` variable. This is another example of how you can manually set configuration values at runtime.

We used the `merge` function, first with the default config file and then with the file selected based on the given environment (e.g.: `./config/Testing.toml`).

Finally, we used `merge` to override any already-set values with provided environment variables. This is important because you likely don’t want your database credentials in a checked-in file. Another way to accomplish this (albeit also not the most secure way) is to set the values via an environment variable.

The `Environment::with_prefix` option ensures that any environment variable that is prefixed with `EA` and matches one of our config paths is used. The separator is set to __ to avoid potential problems related to variable names that include a _. In this case `EA_SERVER__PORT` will set the `server.port` property to the given value.

And that’s it. Calling `Settings::new()` will attempt to load and parse the configuration, giving us a fully filled-out settings object based on the environment chosen.

## Configuring a web application

The next steps is to integrate this into a simple web application.

We’ll demonstrate two ways to propagate this configuration through the application. First, we’ll simply provide it as a global variable. We’ll also walk through passing it around directly to our handlers.

Global variables are fine for small applications, but once things get big and involved, there might be coupling issues and you will likely move to a more explicit way of sending configuration values around.

Let’s spin up a warp web server with two routes, one for each of the options outlined above.


```rust
#[macro_use]
extern crate lazy_static;
use std::convert::Infallible;
use warp::{Filter, Rejection, Reply};

mod settings;

lazy_static! {
    static ref CONFIG: settings::Settings =
        settings::Settings::new().expect("config can be loaded");
}

#[tokio::main]
async fn main() {
    let cfg_route = warp::path("local")
        .and(with_cfg(CONFIG.clone()))
        .and_then(cfg_handler);

    let global_cfg_route = warp::path("global").and_then(global_cfg_handler);

    println!(
        "Server started at localhost:{} and ENV: {}",
        CONFIG.server.port, CONFIG.env
    );

    warp::serve(cfg_route.or(global_cfg_route))
        .run(([0, 0, 0, 0], CONFIG.server.port))
        .await;
}

async fn cfg_handler(cfg: settings::Settings) -> Result<impl Reply, Rejection> {
    Ok(format!(
        "Running on port: {} with url: {}",
        cfg.server.port, cfg.server.url
    ))
}

async fn global_cfg_handler() -> Result<impl Reply, Rejection> {
    Ok(format!(
        "Running with interval: {} and rules: {:?}",
        CONFIG.job.interval, CONFIG.rules
    ))
}

fn with_cfg(
    cfg: settings::Settings,
) -> impl Filter<Extract = (settings::Settings,), Error = Infallible> + Clone {
    warp::any().map(move || cfg.clone())
}
```

Wooosh! So much code! Let’s step through it block by block.

Initially, we used the `lazy_static` crate to lazily create a static version of our config in `settings`. This static reference can be used anywhere in the application; this is the global variant.

The local variant involved the `with_cfg` warp filter at the bottom of the snippet, where we enabled a route to be passed a `settings::Settings` object. This way, the handler explicitly gets the configuration instead of relying on the global state being available.

In the main function, we created the two routes and started the server based on the configured port. As you can see in the `cfg_handler` and `global_cfg_handler`, accessing the config is rather easy either way.

In practice, you would likely send the whole config object not around the world, but in small chunks. For example, the database config might be passed to the database pool for initialization, but the DB pool won’t care about your server port.

You might have noticed that in the global handler, we used `CONFIG.job.interval`, which is a key we didn’t define initially. That’s the next feature we’ll add to our application: a second, completely separate configuration structure.

## Advanced features

Depending on the complexity of your application, it might make sense to have not just one set of configuration files, but multiple sets. They could be differentiated by the domain they handle or the simple fact that some of the config is provided out of the box and some of it needs to be defined by the user.

In any case, it would be nice to be able to use multiple config files, so let’s see if we can make that happen. Luckily, the `config` crate’s `merge` functionality is very powerful and essentially lets us nest and merge as much as we want.

Let’s say we want to add some job configuration for our cron jobs. We want that configuration to be JSON and located in a folder called `./configjson`.

First, we’ll define configuration files for each of our environments.

`Default.json`:
```json
{
    "job": {
        "interval": 1000,
        "enabled": false
    }
}
```
`Development.json`:
```json
{
    "job": {
        "interval": 10
    }
}
```
`Testing.json`:
```json
{}
```
`Production.json`:
```json
{
    "job": {
        "interval": 100000,
        "enabled": true
    }
}
```

Again, we define a default configuration. We can change it, or parts of it, for different environments hierarchically.

Next, we add the fields to the `Settings` struct.

```rust
#[derive(Debug, Deserialize, Clone)]
pub struct Job {
    pub interval: i64,
    pub enabled: bool,
}

#[derive(Debug, Deserialize, Clone)]
pub struct Settings {
    pub server: Server,
    pub rules: Vec<Rule>,
    pub log: Log,
    pub job: Job,
    pub env: ENV,
}
```

Finally, we merge it on top of the existing configuration.

```rust
const OTHER_CONFIG_FILE_PATH: &str = "./configjson/Default.json";
const OTHER_CONFIG_FILE_PREFIX: &str = "./configjson/";

impl Settings {
    pub fn new() -> Result<Self, ConfigError> {
...
        s.merge(File::with_name(&format!("{}{}", CONFIG_FILE_PREFIX, env)))?;

        s.merge(File::with_name(OTHER_CONFIG_FILE_PATH))?;
        s.merge(File::with_name(&format!(
            "{}{}",
            OTHER_CONFIG_FILE_PREFIX, env
        )))?;
...
    }
}
```

Take care with the order of merging. In this case, the `configjson` config will overwrite anything before it. This is fine for our purposes since it defines new keys anyway.

Let’s give it a spin to see if it works using `cargo run` and `cURL`.

First, we’ll use the default configuration (i.e., fall back to `Development`).

```bash
cargo run
Server started at localhost:8080 and ENV: Development

curl http://localhost:8080/local
Running on port: 8080 with url: http://localhost:8080

curl http://localhost:8080/global
Running with interval: 10 and rules: [Rule { name: "all", rule_set: [">5", "<100"] }, Rule { name: "none", rule_set: [] }]
```

Next, we’ll explicitly set `Production`.

```bash
RUN_ENV=Production cargo run
Server started at localhost:8080 and ENV: Production

curl http://localhost:8080/local
Running on port: 8080 with url: http://example.org

curl http://localhost:8080/global
Running with interval: 100000 and rules: [Rule { name: "all", rule_set: [">5", "<100"] }, Rule { name: "none", rule_set: [] }]
```

And finally, let’s set an environment variable explicitly.

```bash
RUN_ENV=Testing EA_SERVER__PORT=9090 cargo run
Server started at localhost:9090 and ENV: Testing
```

Fantastic - it works exactly as we planned. We can override using the chosen environment and manually using environment variables. Both configuration files work and our values are set correctly.

You can find the full example code on [GitHub](https://github.com/zupzup/rust-web-configuration-example).

## Conclusion

Configuration management is a core concern in any nontrivial web application, and the Rust ecosystem provides everything you need and more.

I have personally used configuration systems in countless languages and frameworks before, and I have to say, the `config` crate nailed this. The basics are easy to grasp and quick to set up, and the crate offers powerful extension options and great flexibility in terms of file formats and data structures.





