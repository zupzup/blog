Often when writing automated tests for parts of distributed systems such as a microservice, one runs into the problem of the service under test calling external web services.

In this post, we'll look at the [mockito](https://github.com/lipanski/mockito) library, which provides a way for mocking such requests in tests, making writing those tests a lot more convenient.

The library is being actively developed. Currently, it's possible to match requests on method, path, query, headers and the body, with the possibility to combine these matchers as well.

## Implementation 

Alright, so we start out by writing our small application first.

First, we simply create a `hyper` server:

```rust
type Error = Box<dyn std::error::Error + Send + Sync + 'static>;
type Result<T> = std::result::Result<T, Error>;
type HttpClient = Client<HttpsConnector<HttpConnector>>;

async fn run_server() -> Result<()> {
    let client = init_client();

    let new_service = make_service_fn(move |_| {
        let client_clone = client.clone();
        async { Ok::<_, Error>(service_fn(move |req| route(req, client_clone.clone()))) }
    });

    let addr = "127.0.0.1:3000".parse().unwrap();
    let server = Server::bind(&addr).serve(new_service);

    println!("Listening on http://{}", addr);
    let res = server.await?;
    Ok(res)
}

#[tokio::main]
async fn main() -> Result<()> {
    run_server().await?;
    Ok(())
}
```

In the `init_client` function, we simply initialize a `hyper` HTTPS client, and `route` takes care of routing requests. We only provide two endpoints `/basic` and `/double`:

```rust
async fn route(req: Request<Body>, client: HttpClient) -> Result<Response<Body>> {
    let mut response = Response::new(Body::empty());

    match (req.method(), req.uri().path()) {
        (&Method::GET, "/basic") => {
            *response.body_mut() = basic(req, &client).await?;
        }
        (&Method::GET, "/double") => {
            *response.body_mut() = double(req, &client).await?;
        }
        _ => {
            *response.status_mut() = StatusCode::NOT_FOUND;
        }
    };
    Ok(response)
}

fn init_client() -> HttpClient {
    let https = HttpsConnector::new();
    Client::builder().build::<_, Body>(https)
}
```

The handlers themselves are not particularly interesting, we simply make HTTP requests and parse the response to a struct. One for Cat Facts and one for Todos:

```rust
#[derive(Serialize, Deserialize)]
struct CatFact {
    text: String,
}

#[derive(Serialize, Deserialize)]
struct TODO {
    title: String,
}

async fn basic(_req: Request<Body>, client: &HttpClient) -> Result<Body> {
    let res = do_get_req(&get_todo_url(), &client).await?;
    let body = to_bytes(res.into_body()).await?;
    let todo: TODO = from_slice(&body)?;
    Ok(todo.title.into())
}

async fn double(_req: Request<Body>, client: &HttpClient) -> Result<Body> {
    let res_todo = do_get_req(&get_todo_url(), &client).await?;
    let body_todo = to_bytes(res_todo.into_body()).await?;
    let todo: TODO = from_slice(&body_todo)?;

    let res_cats = do_get_req(&get_cats_url(), &client).await?;
    let body_cats = to_bytes(res_cats.into_body()).await?;
    let fact: CatFact = from_slice(&body_cats)?;
    Ok(format!("Todo: {}, Cat Fact: {}", todo.title, fact.text).into())
}

async fn do_get_req(uri: &str, client: &HttpClient) -> Result<Response<Body>> {
    let request = Request::builder()
        .method(Method::GET)
        .uri(uri)
        .body(Body::empty())?;
    let res = client.request(request).await?;
    Ok(res)
}
```

In the above snippet, there is a helper function for executing a `GET` request and the two handlers, which execute one and two consecutive requests respectively.

Now comes the interesting part in regards to `mockito`. Above, we use the `get_cats_url` and `get_todo_url` functions to get the URLs of the external services we use. These functions look like this:

```rust
#[cfg(test)]
use mockito;

#[cfg(not(test))]
const CATS_URL: &str = "https://cat-fact.herokuapp.com";

#[cfg(not(test))]
const TODO_URL: &str = "https://jsonplaceholder.typicode.com";

fn get_cats_url() -> String {
    #[cfg(not(test))]
    let url = format!("{}/facts/random", CATS_URL);
    #[cfg(test)]
    let url = format!("{}/facts/random", mockito::server_url());
    url
}

fn get_todo_url() -> String {
    #[cfg(not(test))]
    let url = format!("{}/todos/1", TODO_URL);
    #[cfg(test)]
    let url = format!("{}/todos/1", mockito::server_url());
    url
}
```

In each case, we check if we're in the `test` context and if so, we use the url provided by the `mockito` server, which runs during our tests. Otherwise, we simply use the actual URLs.

Ok, if we would run this app, we would simply see the server starting up and then, calling the two endpoints we'd get a string response with Cat Facts and/or Todos.

Now, let's look at how to write an integration test for this simple web app.

In order to run our server, we need a runtime, where we'll spawn it. Then, we'll wait a short time for the server to come up.

Once this setup is done, we make a request to our server and execute assertions on our results.

Before all that however, we register a `mock()` with `mockito`. All of this looks like the following:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use mockito::mock;
    use tokio::runtime::Runtime;

    #[test]
    fn test_basic() {
        let _mt = mock("GET", "/todos/1")
            .with_status(200)
            .with_header("content-type", "application/json")
            .with_body(r#"{"title": "get another cat"}"#)
            .create();

        let mut rt = Runtime::new().unwrap();
        let client = init_client();

        // start server
        rt.spawn(run_server());

        // wait for server to come up
        std::thread::sleep(std::time::Duration::from_millis(50));

        // make requests
        let req_fut = client.request(
            Request::builder()
                .method(Method::GET)
                .uri("http://localhost:3000/basic")
                .body(Body::empty())
                .unwrap(),
        );
        let res = rt.block_on(req_fut).unwrap();
        let body = rt.block_on(to_bytes(res.into_body())).unwrap();

        assert_eq!(std::str::from_utf8(&body).unwrap(), "get another cat");
    }
```

So in the beginning, we mock every `GET` call to the url `/todos/1` to return a HTTP 200, with a hard-coded JSON response. This means, that every call to the above mentioned `mockito::server_url()` with the URL in the mock will get this response. We could also do more complex mocks here, matching on the URL via regex and much more. 

In this case, after the server is started, we use the runtime's `block_on` function to execute the request and wait for the result - same for reading the body.

Now, let's write a test for our second endpoint:

```rust
#[test]
fn test_double() {
    let mut rt = Runtime::new().unwrap();
    let client = init_client();
    let _mc = mock("GET", "/facts/random")
        .with_status(200)
        .with_header("content-type", "application/json")
        .with_body(r#"{"text": "cats are the best living creatures in the universe"}"#)
        .create();

    let _mt = mock("GET", "/todos/1")
        .with_status(200)
        .with_header("content-type", "application/json")
        .with_body(r#"{"title": "get another cat"}"#)
        .create();

    // start server
    rt.spawn(run_server());

    // wait for server to come up
    std::thread::sleep(std::time::Duration::from_millis(50));

    // make requests
    let req_fut = client.request(
        Request::builder()
            .method(Method::GET)
            .uri("http://localhost:3000/double")
            .body(Body::empty())
            .unwrap(),
    );
    let res = rt.block_on(req_fut).unwrap();
    let body = rt.block_on(to_bytes(res.into_body())).unwrap();

    assert_eq!(
        std::str::from_utf8(&body).unwrap(),
        "Todo: get another cat, Cat Fact: cats are the best living creatures in the universe"
    );
}
```

This test is basically the same, with the difference, that we actually have to mock both URLs (TODOs and Cats). If this kind of integration test is familiar to you, you might have noticed one current limitation of `mockito` - you can only use one `mockito::server_url`.

So if you, for example, had two different services you wanted to mock, which have different base URLs, but the same path, like `example.org/hello` and `hello.org/hello`, you couldn't easily provide different mocks for each of them.

In such cases, you might have the option to match on something else in the request and this is something the library will perhaps support in the future.

The full example code including can be found [here](https://github.com/zupzup/rust-mockito-example).

## Conclusion

For testing web services, or applications in general, which rely on communicating to several external APIs, `mockito` seems like a nice solution to make writing integration tests easier.

While the library is in development and there are still some things missing, it already works really well for the majority of use-cases and the API and setup were very intuitive.

#### Resources

* [Code Example](https://github.com/zupzup/rust-mockito-example)
* [mockito](https://github.com/lipanski/mockito)
* [hyper](https://hyper.rs/)
