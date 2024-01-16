*This post was originally posted on the [LogRocket](https://blog.logrocket.com/using-mongodb-in-a-rust-web-service/) blog on 30.07.2020 and was cross-posted here by the author.*

In a [previous post](https://blog.logrocket.com/create-an-async-crud-web-service-in-rust-with-warp/), we covered how to create a web API in Rust with Postgres. This time around, we’ll look at how to create a similar app with [MongoDB](https://www.mongodb.com/), a very popular and widely used document database.

The underlying infrastructure for this application will be the same as in the aforementioned post to show the similarities and differences when switching from a relational to an object database in a Rust web service.

Luckily, MongoDB can be used either synchronously and asynchronously with the official MongoDB Rust library, which means this example will also be fully asynchronous. We’ll demonstrate this concept by building a CRUD API for books.

## Setup

To follow along, all you need is a reasonably recent Rust installation (1.39+). A way to start a local MongoDB instance, such as [Docker](https://www.docker.com/), and a tool to send HTTP requests, such as [curl](https://curl.haxx.se/) or [Postman](https://www.postman.com/), would also be useful.

First, create a new Rust project.

```bash
cargo new rust-web-mongodb-example
cd rust-web-mongodb-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need.

```toml
[dependencies]
tokio = { version = "0.2", features = ["macros", "rt-threaded"] }
warp = "0.2"
serde = {version = "1.0", features = ["derive"] }
thiserror = "1.0"
chrono = { version = "0.4", features = ["serde"] }
futures = { version = "0.3.4", default-features = false, features = ["async-await"]}
mongodb = "1.0.0"
```

For this example, we’ll use the official [Rust MongoDB Driver](https://github.com/mongodb/mongo-rust-driver). The web application is based on [warp](https://github.com/seanmonstar/warp), and we’ll add a few helper libraries, such as `serde` for handling serialization, `chrono` for dealing with time, and `thiserror` for handling errors.

## Structure

To build this small example application, we’ll create three modules:

* `db` — database access logic
* `error` — errors and HTTP error response handling
* `handler` — CRUD HTTP handlers

Let’s start with the database access and work our way upward. We’ll wire everything together in main at the end.


## Database access

First, let’s create a small abstraction for database access, which we can pass around the application.

```rust
use mongodb::bson::{doc, document::Document, oid::ObjectId, Bson};
use mongodb::{options::ClientOptions, Client, Collection};

const DB_NAME: &str = "booky";
const COLL: &str = "books";

const ID: &str = "_id";
const NAME: &str = "name";
const AUTHOR: &str = "author";
const NUM_PAGES: &str = "num_pages";
const ADDED_AT: &str = "added_at";
const TAGS: &str = "tags";

#[derive(Clone, Debug)]
pub struct DB {
    pub client: Client,
}

impl DB {
...
}
```

Next, we need to establish a way to initialize a MongoDB client.

```rust
impl DB {
    pub async fn init() -> Result<Self> {
        let mut client_options = ClientOptions::parse("mongodb://127.0.0.1:27017").await?;
        client_options.app_name = Some("booky".to_string());
        Ok(Self {
            client: Client::with_options(client_options)?,
        })
    }
}
```

Creating a MongoDB client in Rust is quite simple. Simply parse a connection string to create `ClientOptions`, set an application name, and initialize the client with the options.

Let’s walk through how to make a simple query (finding all books in the database).

```rust
    pub async fn fetch_books(&self) -> Result<Vec<Book>> {
        let mut cursor = self
            .get_collection()
            .find(None, None)
            .await
            .map_err(MongoQueryError)?;

        let mut result: Vec<Book> = Vec::new();
        while let Some(doc) = cursor.next().await {
            result.push(self.doc_to_book(&doc?)?);
        }
        Ok(result)
    }

    fn get_collection(&self) -> Collection {
        self.client.database(DB_NAME).collection(COLL)
    }
```

The first step is to select a `collection` using the `get_collection()` helper. This operation doesn’t actually talk to the database; it just sets the collection context we want to operate in.

Next, execute the `find()` command on this collection. In this case, we’ll pass no `filter` and no other `options`, so both parameters are set to `None`. We simply want all objects.

Awaiting the result of `find` gives us a `Cursor`, which is a `Stream`. This stream can be iterated. Once it’s exhausted, if there is more data, the cursor will automatically fetch the next chunk of data until no more is left.

To get from these documents to actual `Books`

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct Book {
    pub id: String,
    pub name: String,
    pub author: String,
    pub num_pages: usize,
    pub added_at: DateTime<Utc>,
    pub tags: Vec<String>,
}
```

create the `doc_to_book` helper.

```rust
    fn doc_to_book(&self, doc: &Document) -> Result<Book> {
        let id = doc.get_object_id(ID)?;
        let name = doc.get_str(NAME)?;
        let author = doc.get_str(AUTHOR)?;
        let num_pages = doc.get_i32(NUM_PAGES)?;
        let added_at = doc.get_datetime(ADDED_AT)?;
        let tags = doc.get_array(TAGS)?;

        let book = Book {
            id: id.to_hex(),
            name: name.to_owned(),
            author: author.to_owned(),
            num_pages: num_pages as usize,
            added_at: *added_at,
            tags: tags
                .iter()
                .filter_map(|entry| match entry {
                    Bson::String(v) => Some(v.to_owned()),
                    _ => None,
                })
                .collect(),
        };
        Ok(book)
    }
```

This helper tries to parse the fields of the returned document based on the data model we want the document to represent. Any of these conversions can fail, of course.

What’s interesting here is that we can use the different `get_$type(key)` functions on the document to convert the data to the types we need. For example, the official MongoDB library supports `chrono::DateTime<Utc>` for dates, which is very convenient. So we can simply use `doc.get_datetime(key)` to convert a date/time within MongoDB to a `chrono` type.

Another useful helper is ``.get_array(key)``, which, as the name suggests, tries to convert the value at the given key to a Vec. In this case, however, we need to make sure the values are actually `Strings` since, in MongoDB, they could be of different types.

## CRUD handlers

The handlers are rather simple; all they do is receive some input and call the database abstraction.

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct BookRequest {
    pub name: String,
    pub author: String,
    pub num_pages: usize,
    pub tags: Vec<String>,
}

pub async fn books_list_handler(db: DB) -> WebResult<impl Reply> {
    let books = db.fetch_books().await.map_err(|e| reject::custom(e))?;
    Ok(json(&books))
}

pub async fn create_book_handler(body: BookRequest, db: DB) -> WebResult<impl Reply> {
    db.create_book(&body).await.map_err(|e| reject::custom(e))?;
    Ok(StatusCode::CREATED)
}

pub async fn edit_book_handler(id: String, body: BookRequest, db: DB) -> WebResult<impl Reply> {
    db.edit_book(&id, &body)
        .await
        .map_err(|e| reject::custom(e))?;
    Ok(StatusCode::OK)
}

pub async fn delete_book_handler(id: String, db: DB) -> WebResult<impl Reply> {
    db.delete_book(&id).await.map_err(|e| reject::custom(e))?;
    Ok(StatusCode::OK)
}
```

The `BookRequest` type is a data object for creating and editing books. Each handler has a `DB` passed to it via a `warp` filter (more on that in the next section).

Besides that, the handlers simply call their respective functionalities in the database abstraction and return either a success message in the form of an `HTTP 200 OK` or, in the case of `books_list_handler`, a list of the existing books.

In a real-world application, we wouldn’t leak the database object here; we’d transform it into some sort of `BookResponse` object. But this minimal approach should suffice for the purpose of this tutorial.

Since we’re working from the bottom up, the next and final stage is `main`, where we’ll put everything together to create a working Rust web application with MongoDB.


## Putting it all together

In `main`, we’ll first define the shared `Result` types, which were used in handler and `db`.

```rust
type Result<T> = std::result::Result<T, error::Error>;
type WebResult<T> = std::result::Result<T, Rejection>;
```

These are just to distinguish between internal results and results we send down to clients (`WebResult`).

The `error` module, which we won’t go into here, just contains a custom `Error` enum which implements warp’s `Reject` trait so we can handle internal errors and return custom errors to clients.

Back in `main`, the next step is to initialize the database abstraction and define some routes.

```rust
#[tokio::main]
async fn main() -> Result<()> {
    let db = DB::init().await?;

    let book = warp::path("book");

    let book_routes = book
        .and(warp::post())
        .and(warp::body::json())
        .and(with_db(db.clone()))
        .and_then(handler::create_book_handler)
        .or(book
            .and(warp::put())
            .and(warp::path::param())
            .and(warp::body::json())
            .and(with_db(db.clone()))
            .and_then(handler::edit_book_handler))
        .or(book
            .and(warp::delete())
            .and(warp::path::param())
            .and(with_db(db.clone()))
            .and_then(handler::delete_book_handler))
        .or(book
            .and(warp::get())
            .and(with_db(db.clone()))
            .and_then(handler::books_list_handler));

    let routes = book_routes.recover(error::handle_rejection);

    println!("Started on port 8080");
    warp::serve(routes).run(([0, 0, 0, 0], 8080)).await;
    Ok(())
}
```

This part goes exactly how you’d expect it to. First, the database wrapper is initialized. Then, the CRUD routes are defined with their respective HTTP methods, parameters, and body definitions.

The `with_db` filter passes the MongoDB database wrapper to the handlers.

```rust
fn with_db(db: DB) -> impl Filter<Extract = (DB,), Error = Infallible> + Clone {
    warp::any().map(move || db.clone())
}
```

At the end of `main`, the actual web server is started.

Now it’s time to test it!

Start a local MongoDB instance using Docker.

```bash
docker run -d -p 27017:27017 -v `pwd`/data/db:/data/db --name bookydb mongo
```

This starts MongoDB on port `27017` with its data directory set to the `data/db` folder in the directory where the command is executed.

Create a book.

```bash
curl -X POST http://localhost:8080/book -d '{"name": "Good Book", "author": "gGeat Author", "num_pages": 500, "tags": ["fun", "exciting"]}' -H "content-type: application/json"
```

Check to make sure it worked by fetching all books.

```bash
curl http://localhost:8080/book

[{"id":"5f19d95800065210009332ef","name":"Good Book","author":"gGeat Author","num_pages":500,"added_at":"2020-07-23T18:39:20.062Z","tags":["fun","exciting"]}]
```

Now let’s edit the book.

```bash
curl -X PUT http://localhost:8080/book/5f19d95800065210009332ef -d '{"name": "Great Book", "author": "Another Great Author", "num_pages": 320, "tags": ["boring", "long"]}' -H "content-type: application/json"
```

To check that it worked:

```bash
curl http://localhost:8080/book

[{"id":"5f19d95800065210009332ef","name":"Great Book","author":"Another Great Author","num_pages":320,"added_at":"2020-07-23T18:39:54.045Z","tags":["boring","long"]}]
```

Finally, test to make sure the books were successfully deleted.

```bash
curl -X DELETE http://localhost:8080/book/5f19d95800065210009332ef
curl http://localhost:8080/book

[]
```

Fantastic! It works exactly as planned.

The full code for this example can be found on [GitHub](https://github.com/zupzup/rust-web-mongodb-example).

## Conclusion

This use case is another shining example of the quickly maturing Rust web ecosystem. The MongoDB driver seems very strong already, providing support for both the tokio and async_std runtimes and providing full synchronous and asynchronous APIs.

The crate recently hit 1.0.0 and is well-documented. Furthermore, the API is intuitive and it’s actively maintained. All in all, a great job by the MongoDB team. I look forward to seeing the first Rust and MongoDB apps in production.

