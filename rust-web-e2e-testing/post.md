*This post was originally posted on the [LogRocket](https://blog.logrocket.com/end-to-end-testing-for-rust-web-services/) blog on 29.06.2020 and was cross-posted here by the author.*

In this tutorial, we’ll demonstrate how to test a [warp](https://github.com/seanmonstar/warp) web application, focusing on integration tests and, more specifically, tests that actually evaluate the whole system — otherwise known as end-to-end tests.

First, we’ll build a small to-do app that features a database and an external HTTP service call. These are things we have to deal with when writing tests. Along the way, we’ll also walk through how to test warp filters and warp applications in general.

The standard strategy for dealing with external dependencies, such as a database or cache, is to simply mock them away — in other words, to replace them with fake implementations. You can do this by using traits or one of the many [mocking libraries](https://crates.io/search?q=mock) available for Rust.

Mocks are useful, but they are, by definition, not the real thing. If, for example, your mock implementation is buggy or incomplete, you’re likely to miss some edge cases. Using real databases and external web services for automated testing, on the other hand, also comes with problems, such as providing the infrastructure, test reproducibility, and speed.

At the end of the day, your decision of which tests to write will depend on your personal preferences and the nature of the service under review. That’s why it’s good to have several techniques in your toolbox.

To illustrate these concepts, we’ll start by creating a simple, self-written mock solution and gradually move toward the proverbial “real thing,” replacing the mocks with more sophisticated fakes and the real database using the web service and sending an HTTP request to it. Of course, our solution will be fully automated and reproducible.

Let’s get started!

## Setup

To follow along, all you need is a reasonably recent Rust installation (1.39+). [Docker](https://www.docker.com/), or some other means of running a [Postgres](https://www.postgresql.org/) database, would also be useful.

First, create a new Rust project.

```bash
cargo new rust-web-e2e-testing
cd rust-web-e2e-testing
```

Edit the `Cargo.toml` file and add the dependencies you’ll need.

```toml
[dependencies]
tokio = { version = "0.2.21", features = ["macros", "rt-threaded", "sync", "time"] }
warp = "0.2.3"
mobc = "0.5.11"
mobc-postgres = { version = "0.5.0" }
hyper = "0.13"
hyper-tls = "0.4.1"
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0.53"
thiserror = "1.0"

[dev-dependencies]
wiremock = "0.2.2"
lazy_static = "=1.4.0"
```

Since we’re using warp with Tokio and building a JSON web service with a database and an external web service, most of these dependencies shouldn’t come as a surprise.

The `dev-dependencies` are a little more interesting, especially [wiremock](https://github.com/LukeMathWalker/wiremock-rs), a well-written library for simulating external web services. This makes it possible to define expected responses while still having the application execute a real HTTP request.

## Writing a web service to test

Let’s start by writing the web service. We’ll move through this section rather quickly, but we covered the basic concepts in a [previous post](https://blog.logrocket.com/create-an-async-crud-web-service-in-rust-with-warp/). We’ll go from the bottom up with the database abstraction.

First, define the `DBAccessor` trait, which will be what we pass around the system to access the database.

```rust
#[async_trait]
pub trait DBAccessor: Send + Sync + Clone + 'static {
    async fn fetch_todos(&self) -> Result<Vec<Todo>>;
    async fn create_todo(&self, name: String) -> Result<Todo>;
}
```

The `async_trait` macro enables us use async in the trait definition, which will work in the future but hasn’t been stabilized in Rust yet.

Next comes the concrete implementation, `DBAccess`, which includes some helpers for getting a connection from the pool and deserializing a database row into a `Todo`.

```rust
#[derive(Deserialize)]
pub struct Todo {
    pub id: i32,
    pub name: String,
    pub checked: bool,
}

#[derive(Clone)]
pub struct DBAccess {
    pub db_pool: DBPool,
}

const INIT_SQL: &str = "./db.sql";

pub fn create_pool() -> std::result::Result<DBPool, mobc::Error<Error>> {
    let config = Config::from_str("postgres://postgres@127.0.0.1:7878/postgres")?;
    Ok(Pool::builder().build(PgConnectionManager::new(config, NoTls)))
}

impl DBAccess {
    pub fn new(db_pool: DBPool) -> Self {
        Self { db_pool }
    }

    pub async fn init_db(&self) -> Result<()> {
        let init_file = fs::read_to_string(INIT_SQL)?;
        let con = self.get_db_con().await?;
        con.batch_execute(init_file.as_str())
            .await
            .map_err(DBInitError)?;
        Ok(())
    }

    async fn get_db_con(&self) -> Result<DBCon> {
        self.db_pool.get().await.map_err(DBPoolError)
    }

    fn row_to_todo(&self, row: &Row) -> Todo {
        let id: i32 = row.get(0);
        let name: String = row.get(1);
        let checked: bool = row.get(2);
        Todo { id, name, checked }
    }
}
```

The db.sql file for creating the table is also very simple.

```bash
CREATE TABLE IF NOT EXISTS todo
(
    id SERIAL PRIMARY KEY NOT NULL,
    name TEXT,
    checked boolean DEFAULT false
);
```

Lastly, implement the DBAccessor trait.

```rust
#[async_trait]
impl DBAccessor for DBAccess {
    async fn fetch_todos(&self) -> Result<Vec<Todo>> {
        let con = self.get_db_con().await?;
        let query = "SELECT id, name, checked FROM todo ORDER BY id ASC";
        let q = con.query(query, &[]).await;
        let rows = q.map_err(DBQueryError)?;

        Ok(rows.iter().map(|r| self.row_to_todo(&r)).collect())
    }

    async fn create_todo(&self, name: String) -> Result<Todo> {
        let con = self.get_db_con().await?;
        let query = "INSERT INTO todo (name) VALUES ($1) RETURNING *";
        let row = con.query_one(query, &[&name]).await.map_err(DBQueryError)?;
        Ok(self.row_to_todo(&row))
    }
}
```

The `db` module enables us to connect to and initialize the database and to create and fetch todos.

In this application, we want to create todos with random cat facts from this [cat facts app](http://cat-fact.herokuapp.com/#/), so the next module we’ll need is an HTTP abstraction.

Again, start with a trait:

```rust
#[async_trait]
pub trait HttpClient: Send + Sync + Clone + 'static {
    async fn get_cat_fact(&self) -> Result<String>;
}
```

The HttpClient is a simpler. The same goes for the concrete Client implementation.

```rust
#[derive(Clone)]
pub struct Client {
    client: HyperClient<HttpsConnector<HttpConnector>>,
}

#[derive(Debug, Deserialize)]
pub struct CatFact {
    pub text: String,
}

impl Client {
    pub fn new() -> Self {
        let HTTPs = HttpsConnector::new();
        Self {
            client: HyperClient::builder().build::<_, Body>(https),
        }
    }

    fn get_url(&self) -> String {
        URI.to_owned()
    }
}

#[async_trait]
impl HttpClient for Client {
    async fn get_cat_fact(&self) -> Result<String> {
        let req = Request::builder()
            .method(Method::GET)
            .uri(&format!("{}{}", self.get_url(), "/facts/random"))
            .header("content-type", "application/json")
            .header("accept", "application/json")
            .body(Body::empty())?;
        let res = self.client.request(req).await?;
        if !res.status().is_success() {
            return Err(error::Error::GetCatFactError(res.status()));
        }
        let body_bytes = to_bytes(res.into_body()).await?;
        let json = from_slice::<CatFact>(&body_bytes)?;
        Ok(json.text)
    }
}
```

This implementation uses the great [hyper](http://hyper.rs/) HTTP client to get a new, interesting `CatFact` and returns only the actual fact as text, dismissing the metadata.

With these two abstractions out of the way, we can move up one level and implement the two handlers for fetching and creating todos.

The handler module features these two functions:

```rust
#[derive(Serialize)]
pub struct TodoResponse {
    pub id: i32,
    pub name: String,
    pub checked: bool,
}

impl TodoResponse {
    pub fn of(todo: Todo) -> TodoResponse {
        TodoResponse {
            id: todo.id,
            name: todo.name,
            checked: todo.checked,
        }
    }
}

pub async fn list_todos_handler(db_access: impl DBAccessor) -> Result<impl Reply> {
    let todos = db_access
        .fetch_todos()
        .await
        .map_err(|e| reject::custom(e))?;
    Ok(json::<Vec<_>>(
        &todos.into_iter().map(|t| TodoResponse::of(t)).collect(),
    ))
}

pub async fn create_todo(
    http_client: impl HttpClient,
    db_access: impl DBAccessor,
) -> Result<impl Reply> {
    let cat_fact = http_client
        .get_cat_fact()
        .await
        .map_err(|e| reject::custom(e))?;
    Ok(json(&TodoResponse::of(
        db_access
            .create_todo(cat_fact)
            .await
            .map_err(|e| reject::custom(e))?,
    )))
}
```

In the above snippet, a `TodoResponse` is defined, which is the JSON representation of todos from the database. The list handler simply queries the database and returns all entries to the caller.

In the `create_todo` handler, we use both our database and HTTP abstractions to fetch a new cat fact, create a todo from it, and return the newly created todo.

In both cases, all errors are handled and sent down to the client properly.

Speaking of errors, there were some custom error types in the above code that we’ve yet to go over. Let’s look at the `error` module, which is responsible for error handling throughout the application, next.

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("error getting connection from DB pool: {0}")]
    DBPoolError(mobc::Error<tokio_postgres::Error>),
    #[error("error executing DB query: {0}")]
    DBQueryError(#[from] tokio_postgres::Error),
    #[error("error creating table: {0}")]
    DBInitError(tokio_postgres::Error),
    #[error("error reading file: {0}")]
    ReadFileError(#[from] std::io::Error),
    #[error("http client error: {0}")]
    HyperHttpError(#[from] hyper::http::Error),
    #[error("http client error: {0}")]
    HypeError(#[from] hyper::error::Error),
    #[error("http client error: {0}")]
    JSONError(#[from] serde_json::error::Error),
    #[error("http client error: {0}")]
    GetCatFactError(StatusCode),
}

#[derive(Serialize)]
struct ErrorResponse {
    message: String,
}

impl warp::reject::Reject for Error {}

pub async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
    let code;
    let message;

    if err.is_not_found() {
        code = StatusCode::NOT_FOUND;
        message = "Not Found";
    } else if let Some(_) = err.find::<warp::filters::body::BodyDeserializeError>() {
        code = StatusCode::BAD_REQUEST;
        message = "Invalid Body";
    } else if let Some(e) = err.find::<Error>() {
        match e {
            Error::DBQueryError(_) => {
                eprintln!("{}", e);
                code = StatusCode::BAD_REQUEST;
                message = "DB Error: Could not Execute request";
            }
            _ => {
                eprintln!("unhandled application error: {:?}", err);
                code = StatusCode::INTERNAL_SERVER_ERROR;
                message = "Internal Server Error";
            }
        }
    } else if let Some(_) = err.find::<warp::reject::MethodNotAllowed>() {
        code = StatusCode::METHOD_NOT_ALLOWED;
        message = "Method Not Allowed";
    } else {
        eprintln!("unhandled error: {:?}", err);
        code = StatusCode::INTERNAL_SERVER_ERROR;
        message = "Internal Server Error";
    }

    let json = warp::reply::json(&ErrorResponse {
        message: message.into(),
    });

    Ok(warp::reply::with_status(json, code))
}
```

This is just some boilerplate of how to deal with warp `Rejections`, so we can keep the rest of the code relatively clean with custom errors instead of strings.

Let’s finish the web application by looking at `main`, where it’s all put together.

First, we need two warp filters to pass the database and HTTP abstractions to the handlers.

```rust
fn with_db(
    db_access: impl db::DBAccessor,
) -> impl Filter<Extract = (impl db::DBAccessor,), Error = Infallible> + Clone {
    warp::any().map(move || db_access.clone())
}

fn with_http_client(
    http_client: impl HttpClient,
) -> impl Filter<Extract = (impl HttpClient,), Error = Infallible> + Clone {
    warp::any().map(move || http_client.clone())
}
```

Then we can define the routes for the application.

```rust
fn router(
    http_client: impl http::HttpClient,
    db_access: impl db::DBAccessor,
) -> impl Filter<Extract = impl Reply, Error = Infallible> + Clone {
    let todo = warp::path("todo");
    let todo_routes = todo
        .and(warp::get())
        .and(with_db(db_access.clone()))
        .and_then(handler::list_todos_handler)
        .or(todo
            .and(warp::post())
            .and(with_http_client(http_client.clone()))
            .and(with_db(db_access.clone()))
            .and_then(handler::create_todo));

    todo_routes.recover(error::handle_rejection)
}
```

We put it all together into a run function, which is called by main. For reasons we’ll discuss later, this is put into its own function:

```rust
type Result<T> = std::result::Result<T, Rejection>;
type DBCon = Connection<PgConnectionManager<NoTls>>;
type DBPool = Pool<PgConnectionManager<NoTls>>;

#[tokio::main]
async fn main() {
    run().await;
}

async fn run() {
    let db_pool = db::create_pool().expect("database pool can be created");
    let db_access = db::DBAccess::new(db_pool);

    db_access
        .init_db()
        .await
        .expect("database can be initialized");
    let http_client = http::Client::new();

    println!("Server started at localhost:8080");
    warp::serve(router(http_client, db_access))
        .run(([0, 0, 0, 0], 8080))
        .await;
}
```

Running this application starts an HTTP server on port 8080. We can create and list todos with the following commands.


```bash
curl http://localhost:8080/todo
curl -X POST http://localhost:8080/todo
```

The full code of this example is available on [GitHub](https://github.com/zupzup/rust-web-e2e-testing).

With this rather long setup out of the way, let’s start testing!

## Testing with mocking

In Rust, it’s common to write unit tests directly inside the file being tested and integration tests inside a `tests` folder outside of `src`.

However, since the integration testing approach is its own build entity and is meant to test the public interfaces of the crate, this isn’t very useful for what we want to do. We’ll take more of a hybrid approach.

To start our mock-to-real journey, let’s create a `test` module inside `src` and add it to `main.rs`.

```rust
#[cfg(test)]
mod tests;
```

This means the module is only loaded for tests and won’t show up in our binary.

Since the plan is to start out with very basic mocks, let’s build those first in `tests/mock.rs`.

```rust
use crate::db::{DBAccessor, Todo};
use crate::HTTP::HttpClient;
...

#[derive(Clone)]
pub struct MockHttpClient {}
type Result<T> = std::result::Result<T, error::Error>;

#[async_trait]
impl HttpClient for MockHttpClient {
    async fn get_cat_fact(&self) -> Result<String> {
        Ok(String::from("cat fact"))
    }
}

#[derive(Clone)]
pub struct MockDBAccessor {}

#[async_trait]
impl DBAccessor for MockDBAccessor {
    async fn fetch_todos(&self) -> Result<Vec<Todo>> {
        Ok(vec![Todo {
            id: 1,
            name: String::from("first todo"),
            checked: true,
        }])
    }

    async fn create_todo(&self, name: String) -> Result<Todo> {
        Ok(Todo {
            id: 2,
            name: name,
            checked: false,
        })
    }
}
```

Here we simply implemented two hardcoded versions of the database and HTTP abstractions defined in our application. Now we can write simple tests for our two handlers.

```rust
#[tokio::test]
async fn test_list_todos_mock() {
    let r = router(MockHttpClient {}, MockDBAccessor {});
    let resp = request().path("/todo").reply(&r).await;
    assert_eq!(resp.status(), 200);
    assert_eq!(
        resp.body(),
        r#"[{"id":1,"name":"first todo","checked":true}]"#
    );
}

#[tokio::test]
async fn test_create_todo_mock() {
    let r = router(MockHttpClient {}, MockDBAccessor {});
    let resp = request()
        .path("/todo")
        .method("POST")
        .body("")
        .reply(&r)
        .await;
    assert_eq!(resp.status(), 200);
    assert_eq!(resp.body(), r#"{"id":2,"name":"cat fact","checked":false}"#);
}
```

What’s going on here? The first thing to note is the use of the fantastic use `warp::test` utilities to test warp filters and handlers. These tools enable you to create HTTP requests against a warp test harness. Since warp `Filters` are just functions and handlers are just futures, you can actually plug in the whole router or other custom filters to test what comes out the other side. This can be very useful for testing complex filters.

In the above example, however, we usef the `reply` method with our router, instantiated with the two basic mocks, which will give us an actual HTTP response on which we can assert. In this case, we should make sure the response matches the hardcoded data we returned in the basic mocks.

With a powerful mocking library, you could go pretty far with this approach, dynamically mocking all kinds of responses and validating that everything works as you expect.

However, we’d like to also see if our HTTP abstraction works as planned. For the next step, we’ll send actual HTTP requests.

## A hybrid approach

To send real HTTP requests (without depending on external services being up and running or spamming them), we’ll use the wiremock library.

First, build a thin wrapper around it so you can use it easily throughout the application and in your tests.

```rust
pub struct WiremockServer {
    pub server: Option<MockServer>,
}

impl WiremockServer {
    pub fn new() -> Self {
        Self { server: None }
    }

    pub async fn init(&mut self) {
        let mock_server = MockServer::start().await;
        Mock::given(method("GET"))
            .and(path("/facts/random"))
            .respond_with(
                ResponseTemplate::new(200).set_body_string(r#"{"text": "wiremock cat fact"}"#),
            )
            .mount(&mock_server)
            .await;
        self.server = Some(mock_server);
    }
}

lazy_static! {
    pub static ref MOCK_HTTP_SERVER: RwLock<WiremockServer> = RwLock::new(WiremockServer::new());
}

async fn setup_wiremock() {
    MOCK_HTTP_SERVER.write().unwrap().init().await;
}
```

This simple construction allows us to spawn a wiremock server, which searches for a random port on your system to start on, on demand and access it from anywhere.

You’ll also notice that we registered a `Mock` for the `/facts/given` endpoint. This means all requests to this path on the wiremock server will return the provided response. wiremock can do a lot more with custom matchers for all parts of the request you want to replace.

During the setup stage, you might have wondered about the `get_url` helper in the HTTP abstraction, which, in that form, seemed quite useless. We’ll extend it during this step.

```rust
     fn get_url(&self) -> String {
        #[cfg(not(test))]
        return URI.to_owned();
        #[cfg(test)]
        return match crate::tests::MOCK_HTTP_SERVER.read().unwrap().server {
            Some(ref v) => v.uri(),
            None => URI.to_owned(),
        };
    }
```

This enables us to see whether a wiremock server is running during a test run. If it is, we can use its URI for the requests and fall back to the standard URI otherwise.

This is necessary because we don’t know beforehand which port the wiremock server will start on. There are other ways to do this, but this is a simple approach to help you get started.

Now we can write tests that send actual HTTP requests to wiremock.

```rust
#[tokio::test]
async fn test_create_and_list_todo_hybrid() {
    setup_wiremock().await;
    let r = router(http::Client::new(), MockDBAccessor {});
    let resp = request()
        .path("/todo")
        .method("POST")
        .body("")
        .reply(&r)
        .await;
    assert_eq!(resp.status(), 200);
    assert_eq!(
        resp.body(),
        r#"{"id":2,"name":"wiremock cat fact","checked":false}"#
    );

    let resp = request().path("/todo").reply(&r).await;
    assert_eq!(resp.status(), 200);
    assert_eq!(
        resp.body(),
        r#"[{"id":1,"name":"first todo","checked":true}]"#
    );
}
```

As you can see, the tests are very similar. The main difference is that we created a real HTTP client that sends real HTTP requests to wiremock. And it works – nice!

But we’re still using a mock for the database and we’d really love to see if our queries work as expected. So let’s fix that.

## Testing with no mocks

To use a real database in your integration tests, you actually have to, well, start a database server.

This may or may not be practical, depending on the database engine you’re using. With a Postgres database, you can simply start an instance locally or on CI (some CI providers also provide Postgres/Redis and other externals as running services during CI runs).

```bash
docker run -p 7878:5432 -d postgres:9.6.12
```

Another thing to keep in mind is that you can only run one of these tests at once. Otherwise, you risk interfering with the database state of other tests. To ensure this, you can set --test-threads=1 when running the test.

```bash
cargo test --offline -- --color=always --test-threads=1 --nocapture
```

If you have a huge number of tests, with the database cleanup step after each test, your tests will take a lot longer using this approach than a mock solution.

The next step is to create a helper function to connect to the database and reset it, since we want a fresh state for each test.

```rust
async fn init_db() -> impl db::DBAccessor {
    let db_pool = db::create_pool().expect("database pool can be created");
    let db_access = db::DBAccess::new(db_pool.clone());

    db_access
        .init_db()
        .await
        .expect("database can be initialized");

    let con = db_pool.get().await.unwrap();
    let query = format!("BEGIN;DELETE FROM todo;ALTER SEQUENCE todo_id_seq RESTART with 1;COMMIT;");
    let _ = con.batch_execute(query.as_str()).await;

    db_access
}
```

Create the connection pool and `DBAccess` instance, then delete all data and reset the id sequence (so you have a fresh start each time) and return the instance to be used in a test.

With this in place, we can write a test that uses a real database and sends real HTTP requests.

```rust
#[tokio::test]
async fn test_create_and_list_full() {
    setup_wiremock().await;
    let r = router(http::Client::new(), init_db().await);
    let resp = request()
        .path("/todo")
        .method("POST")
        .body("")
        .reply(&r)
        .await;
    assert_eq!(resp.status(), 200);
    assert_eq!(
        resp.body(),
        r#"{"id":1,"name":"wiremock cat fact","checked":false}"#
    );

    let resp = request().path("/todo").reply(&r).await;
    assert_eq!(resp.status(), 200);
    assert_eq!(
        resp.body(),
        r#"[{"id":1,"name":"wiremock cat fact","checked":false}]"#
    );
}
```

We could stop here, but we want to go full end-to-end — which means we don’t want to use the warp testing harness, but our actual web service.

Let’s wrap up this tutorial with some end-to-end fireworks!

## Full end-to-end testing

Remember when I said we will see later why everything is executed in the run function? This is the time for that. To test the running web service with the all the real things, we need a small utility to run the server:

```rust
pub struct Server {
    pub started: AtomicBool,
}

impl Server {
    pub fn new() -> Server {
        Server {
            started: AtomicBool::new(false),
        }
    }

    pub async fn init_server(&mut self) {
        if !self.started.load(Ordering::Relaxed) {
            thread::spawn(move || {
                let rt = tokio::runtime::Runtime::new().expect("runtime starts");
                rt.spawn(run());
                loop {
                    thread::sleep(Duration::from_millis(100_000));
                }
            });
            delay_for(Duration::from_millis(100)).await;
            self.started.store(true, Ordering::Relaxed);
        }
    }
}

lazy_static! {
    pub static ref MOCK_HTTP_SERVER: RwLock<WiremockServer> = RwLock::new(WiremockServer::new());
    static ref SERVER: RwLock<Server> = RwLock::new(Server::new());
}

async fn init_real_server() {
    let _ = init_db().await;
    SERVER.write().unwrap().init_server().await;
}
```

Basically, the idea is to start the service once and to reuse that instance for multiple test runs, since the service itself is stateless.

For this purpose, we’ll create the Server abstraction, which has an `init_server` method. This starts the crate’s run method on the first call. It also starts the warp web service and sets `started` to true so we can’t accidentally start it again.

For this to work, the server itself is started in a separate thread on a new Tokio runtime while the thread sleeps forever. There may be a short delay before the server is up, depending on the amount of setup the service must undertake before it can respond. This only happens once.

For the tests, we’ll also create an `init_real_server` helper, which clears the database and ensures that the server is up.

Since we’re now running the web service, we need to send our HTTP requests from the outside, which requires an HTTP client. Luckily, we have `hyper`.

```rust
fn http_client() -> HyperClient<HttpsConnector<HttpConnector>> {
    let https = HttpsConnector::new();
    HyperClient::builder().build::<_, Body>(https)
}
```

Here’s what the actual end-to-end tests should look like:

```rust
#[tokio::test]
async fn test_create_and_list_e2e() {
    setup_wiremock().await;
    init_real_server().await;
    let http_client = http_client();

    let req = Request::builder()
        .method(Method::POST)
        .uri("http://localhost:8080/todo")
        .body(Body::empty())
        .unwrap();
    let resp = http_client.request(req).await.unwrap();
    assert_eq!(resp.status(), 200);
    let body_bytes = to_bytes(resp.into_body()).await.unwrap();
    assert_eq!(
        body_bytes,
        r#"{"id":1,"name":"wiremock cat fact","checked":false}"#
    );

    let req = Request::builder()
        .method(Method::GET)
        .uri("http://localhost:8080/todo")
        .body(Body::empty())
        .unwrap();
    let resp = http_client.request(req).await.unwrap();
    assert_eq!(resp.status(), 200);
    let body_bytes = to_bytes(resp.into_body()).await.unwrap();
    assert_eq!(
        body_bytes,
        r#"[{"id":1,"name":"wiremock cat fact","checked":false}]"#
    );
}

#[tokio::test]
async fn test_list_e2e() {
    setup_wiremock().await;
    init_real_server().await;
    let http_client = http_client();

    let req = Request::builder()
        .method(Method::GET)
        .uri("http://localhost:8080/todo")
        .body(Body::empty())
        .unwrap();
    let resp = http_client.request(req).await.unwrap();
    assert_eq!(resp.status(), 200);
    let body_bytes = to_bytes(resp.into_body()).await.unwrap();
    assert_eq!(body_bytes, r#"[]"#);
}
```

Here we set up wiremock, initialized the database and server, and sent an HTTP request to `http://localhost:8080`, which is the URL our web service runs on.

The second test is there to show how this works with multiple tests. As you can see, the database is reset every time and the server is reused.

What a journey! You can find the full code for this example on [GitHub](https://github.com/zupzup/rust-web-e2e-testing).

## Conclusion

Testing, especially in the context of distributed web services, is a complex topic. There are no definitive answers regarding how to test, no one-size-fits-all approach. Usually, a healthy mix of unit, integration, and end-to-end testing is a good place to start, but there are certainly cases in which one or more of these approaches impractical.

For this reason, I believe it’s important to be comfortable with multiple testing techniques and to understand the pros, cons, and tradeoffs associated with each.

If you made it through this tutorial, you should now know a few integration testing techniques using external dependencies, ranging from very low-effort mocks to automatically testing the fully integrated application.

We kept it pretty low-tech with few libraries, writing many of the things by hand, which hopefully showed that none of this is magic — or, quite frankly, even particularly complex - under the hood.

