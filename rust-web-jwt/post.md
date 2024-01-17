*This post was originally posted on the [LogRocket](https://blog.logrocket.com/jwt-authentication-in-rust/) blog on 29.10.2020 and was cross-posted here by the author.*

[JSON Web Tokens (JWTs)](https://jwt.io/) are a standard for securely representing attributes or claims between systems. They can be used in a client-server fashion to enable stateless authorization, whereas cookies are inherently stateful.

However, they are more flexible than that and can also be used in myriad other ways. A prominent use case is secure user state propagation in a microservice architecture. In such a setup, the use case of JWTs can be purely limited to the backend side, with a stateful authorization mechanism toward the frontend. Upon logging in, a session token is mapped onto a JWT, which is then used within the microservice cluster to authorize requests (access control), but also to distribute state about the user (information distribution).

The advantage of this is that other services, or clients, don’t need to refetch information, which is stored within the JWT. For example, a user role, the user email, or whatever you need to access regularly can be encoded inside a JWT. And because JWTs are cryptographically signed, the data stored within them is secure and can’t be manipulated easily.

In this tutorial, we’ll explain how to implement authentication and authorization using JWTs in a Rust web application. We won’t go into very much detail on JWTs themselves; there are [great resources](https://jwt.io/introduction/) on that topic already.

The example we’ll build will focus more on the access control part of JWTs, so we’ll only save the user ID and the user’s role inside the token — everything we need to make sure a user is allowed to access a resource.

As is custom for security-related blog posts, here is a short disclaimer: The code shown in this blog post is not production ready and shouldn’t be copy/pasted. The sole aim of this example is to show off some of the concepts, techniques, and libraries you might want to use when building an authentication/authorization system.

With that out of the way, let’s get started!

## Setup

To follow along, you’ll need a recent Rust installation (1.39+) and a tool to send HTTP requests, such as cURL.

First, create a new Rust project.

```bash
cargo new rust-jwt-example
cd rust-jwt-example
```

Next, edit the Cargo.toml file and add the dependencies you’ll need.

```toml
[dependencies]
jsonwebtoken = "=7.2"
tokio = { version = "0.2", features = ["macros", "rt-threaded", "sync", "time"] }
warp = "0.2"
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0"
thiserror = "1.0"
chrono = "0.4"
```

We’ll build the web application using the lightweight warp library, which uses tokio as its async runtime. We’ll use Serde for JSON handling and Thiserror and Chrono to handle errors and dates, respectively.

To deal with the JSON Web Tokens, we’ll use the aptly named [jsonwebtoken](https://crates.io/crates/jsonwebtoken) crate, which is mature and widely used within the Rust ecosystem.

## Web server

We’ll start by creating a simple web server with a couple of endpoints and an in-memory user store. In a real application, we would probably have a database for user storage. But since that’s not important for our example, we’ll simply hardcode them in memory.

```rust
type Result<T> = std::result::Result<T, error::Error>;
type WebResult<T> = std::result::Result<T, Rejection>;
type Users = Arc<RwLock<HashMap<String, User>>>;
```

Here we define two helper types for `Result`, specifying an internal result type for propagating errors throughout the application and an external result type for sending errors to the caller.

We also define the `Users` type, which is a shared `HashMap`. This is our in-memory user store and we can initialize it like this:

```rust
mod auth;
mod error;

#[derive(Clone)]
pub struct User {
    pub uid: String,
    pub email: String,
    pub pw: String,
    pub role: String,
}

#[tokio::main]
async fn main() {
    let users = Arc::new(RwLock::new(init_users()));
    ...
}

fn init_users() -> HashMap<String, User> {
    let mut map = HashMap::new();
    map.insert(
        String::from("1"),
        User {
            uid: String::from("1"),
            email: String::from("user@userland.com"),
            pw: String::from("1234"),
            role: String::from("User"),
        },
    );
    map.insert(
        String::from("2"),
        User {
            uid: String::from("2"),
            email: String::from("admin@adminaty.com"),
            pw: String::from("4321"),
            role: String::from("Admin"),
        },
    );
    map
}
```

We use a `HashMap`, which enables us to easily search by the user’s ID. The map is wrapped in an `RwLock` because multiple threads can access the users map at the same time. This is also the reason it’s finally put into an `Arc` - an atomic, reference counted smart pointer - which enables us to share this map between threads.

Since we’re building an asynchronous web service and we can’t know in advance on which threads our handler futures will run, we need to make everything we pass around thread-safe.

We’ll set the users map with two users: one with role `User` and one with role `Admin`. Later on, we’ll create endpoints, which can only be accessed with the `Admin` role. This way, we can test that our authorization logic works as intended.

Since we’re using warp, we also need to build a filter to pass the users map to endpoints.

```rust
fn with_users(users: Users) -> impl Filter<Extract = (Users,), Error = Infallible> + Clone {
    warp::any().map(move || users.clone())
}
```

With this first bit of setup out of the way, we can define some basic routes and start the web server.

```rust
#[tokio::main]
async fn main() {
    let users = Arc::new(RwLock::new(init_users()));

    let login_route = warp::path!("login")
        .and(warp::post())
        .and_then(login_handler);

    let user_route = warp::path!("user")
        .and_then(user_handler);
    let admin_route = warp::path!("admin")
        .and_then(admin_handler);

    let routes = login_route
        .or(user_route)
        .or(admin_route)
        .recover(error::handle_rejection);

    warp::serve(routes).run(([127, 0, 0, 1], 8000)).await;
}

pub async fn login_handler() -> WebResult<impl Reply> {
    Ok("Login")
}

pub async fn user_handler() -> WebResult<impl Reply> {
    Ok("User")
}

pub async fn admin_handler() -> WebResult<impl Reply> {
    Ok("Admin")
}
```

In the above snippet, we define three handlers:

* `POST /login` — log in with e-mail and password
* `GET /user` — an endpoint for every user
* `GET /admin` — an endpoint only for admins

Don’t worry about `.recover(error::handle_rejection)` yet; we’ll deal with error handling a bit later on.

## Authentication

Let’s build the login functionality so users and admins can authenticate.

The first step is to get the credentials inside the `login_handler`.

```rust
#[derive(Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub pw: String,
}

#[derive(Serialize)]
pub struct LoginResponse {
    pub token: String,
}
```

This is the API we define for the login mechanism. A client sends an email and password and receives a JSON Web Token as response, which the client can then use to make authenticated requests by putting this token inside the `Authorization: Bearer $token` header field.

We define this as a body to the `login_handler`, like this:

```rust
async fn main() {
    ...
    let login_route = warp::path!("login")
        .and(warp::post())
        .and(with_users(users.clone()))
        .and(warp::body::json())
        .and_then(login_handler);
    ...
}
```

In the `login_handler`, the signature and implementation change to:

```rust
pub async fn login_handler(users: Users, body: LoginRequest) -> WebResult<impl Reply> {
    match users.read() {
        Ok(read_handle) => {
            match read_handle
                .iter()
                .find(|(_uid, user)| user.email == body.email && user.pw == body.pw)
            {
                Some((uid, user)) => {
                    let token = auth::create_jwt(&uid, &Role::from_str(&user.role))
                        .map_err(|e| reject::custom(e))?;
                    Ok(reply::json(&LoginResponse { token }))
                }
                None => Err(reject::custom(WrongCredentialsError)),
            }
        }
        Err(_) => Err(reject()),
    }
}
```

What’s happening here? First, we access the shared `Users` map by calling `.read()`, which gives us a read-lock on the map. This is all we need for now.

Then, we iterate over this read-only version of the users map, trying to find a user with the `email` and `pw` as provided in the incoming body.

If we don’t find a user, we return a `WrongCredentialsError`, telling the user they didn’t use valid credentials. Otherwise, we call `auth::create_jwt` with the existing user’s user ID and role, which returns a `token`. This is what we send back to the caller.

Let’s look at the `auth` module next.

In `auth.rs`, we first define some useful data types and constants.

```rust
const BEARER: &str = "Bearer ";
const JWT_SECRET: &[u8] = b"secret";

#[derive(Clone, PartialEq)]
pub enum Role {
    User,
    Admin,
}

impl Role {
    pub fn from_str(role: &str) -> Role {
        match role {
            "Admin" => Role::Admin,
            _ => Role::User,
        }
    }
}

impl fmt::Display for Role {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Role::User => write!(f, "User"),
            Role::Admin => write!(f, "Admin"),
        }
    }
}

#[derive(Debug, Deserialize, Serialize)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
}
```

The `Role` enum is simply a mapping of the `Admin` and User roles, so we don’t have to muck around with strings, which is way too error-prone for security-critical stuff like this.

We also define helper methods to convert from and to strings from the `Role` enum, since this role is saved within the JWT.

Another important type is `Claims`. This is the data we will save inside and expect of our JWTs. The sub depicts the so-called subject, so “who,” in this case. `exp` is the expiration date of the token. We also put the user `role` in there as a custom data point.

The two constants are the prefix of the expected `Authorization` header and the very important `JWT_SECRET`. This is the key with which we sign our JSON Web Tokens. In a real system, this would be a long, securely stored string that is changed regularly. If this secret were to leak, anyone could decode all JWTs created with this secret. You could also use a different secret for each user, for example, which would enable you to easily invalidate all of a user’s tokens in case of a data breach by simply changing this secret.

Let’s look at the `create_jwt` function next.

```rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};

pub fn create_jwt(uid: &str, role: &Role) -> Result<String> {
    let expiration = Utc::now()
        .checked_add_signed(chrono::Duration::seconds(60))
        .expect("valid timestamp")
        .timestamp();

    let claims = Claims {
        sub: uid.to_owned(),
        role: role.to_string(),
        exp: expiration as usize,
    };
    let header = Header::new(Algorithm::HS512);
    encode(&header, &claims, &EncodingKey::from_secret(JWT_SECRET))
        .map_err(|_| Error::JWTTokenCreationError)
}
```

First, we calculate an expiration date for this token. In this case, we only set it to 60 seconds in the future. This is nice for testing because we don’t have to wait long for the token to expire.

The expiration set can be defined using different strategies, but since these tokens are security-critical and hold sensible information, they definitely should expire at some point. Some systems rely on a refresh token mechanism, setting short (minutes/hours) expiration times and providing a refresh token to the caller, which can be used to get a new token if the old one is expired.

Next, we create the `Claims` struct with the user’s ID, the user’s role, and the expiration date. After that comes our first interaction with the `jsonwebtoken` crate.

If you have dealt with JWTs before, you’ll know they consist of three parts:

1. Header
2. Payload
3. Signature

This is reflected here since we create a new header and encode this header, plus our payload (claims) with the above-mentioned secret. If this fails, we return an error. Otherwise, we return the resulting JWT.

Now users can log in to our service, but we don’t have a mechanism for handling authorization yet. We’ll look at that next.

## Authorization

We stay within the `auth.rs` module. Since we’re using warp, the best way to add additional functionality, such as middleware, to our handlers is with a filter.

So we define a `with_auth` filter.

```rust
use warp::{
    filters::header::headers_cloned,
    http::header::{HeaderMap, HeaderValue, AUTHORIZATION},
    reject, Filter, Rejection,
};

pub fn with_auth(role: Role) -> impl Filter<Extract = (String,), Error = Rejection> + Clone {
    headers_cloned()
        .map(move |headers: HeaderMap<HeaderValue>| (role.clone(), headers))
        .and_then(authorize)
}
```

This filter can be added to an endpoint using `.and(with_auth(Role::Admin)`, for example, which would mean that this handler can only be accessed by users with the `Admin` role.

Because, in a real-world system, we would very likely connect to a database, cache, or some other external system in this step, I decided to create an async filter. This isn’t strictlyrequiredneeded in this case, but it will come in handy in any case where the user store isn’t a static, in-memory map.

There are a few steps we need to take to authorize a user:

* Get the `Authorization` header; fail if it isn’t there
* Validate the header, making sure it has a valid format (`Bearer $JWT`); fail if that’s not the case
* Extract the JWT string from the header; fail if that doesn’t work
* Decode the JWT; fail if it’s invalid or expired
* Check the role saved in the JWT and compare it with the given `role`; fail if, for example, the JWT role is User but the endpoint requires `Admin`
* Extract the uid from the JWT, passing it into the decorated handler

That’s quite a few steps! We need to approach error-handling carefully, since any bugs here will lead to severe holes.

In the `with_auth` function above, we use the `headers_cloned()` warp filter to get a copy of the request headers stored inside a map. Then we bundle it together with the `role` and pass it to the `authorize` function, which is the meat of the authorization functionality.

```rust
async fn authorize((role, headers): (Role, HeaderMap<HeaderValue>)) -> WebResult<String> {
    match jwt_from_header(&headers) {
        Ok(jwt) => {
            ...
        }
        Err(e) => return Err(reject::custom(e)),
    }
}
```

Since this is an `async` function, we need to use `and_then` in the filter. As I mentioned above, this isn’t necessary in this example, but in a real-world example, you might pass a handle to an external system in here as well, which you might need for authorization. An example would be a cache or database for mapping session tokens to internal tokens or for fetching some needed metadata.

In this example, we initially call the `jwt_from_header` function with the header map to get the JWT from the `Authorization` header.

```rust
fn jwt_from_header(headers: &HeaderMap<HeaderValue>) -> Result<String> {
    let header = match headers.get(AUTHORIZATION) {
        Some(v) => v,
        None => return Err(Error::NoAuthHeaderError),
    };
    let auth_header = match std::str::from_utf8(header.as_bytes()) {
        Ok(v) => v,
        Err(_) => return Err(Error::NoAuthHeaderError),
    };
    if !auth_header.starts_with(BEARER) {
        return Err(Error::InvalidAuthHeaderError);
    }
    Ok(auth_header.trim_start_matches(BEARER).to_owned())
}
```

This function does the first couple of steps, checking if the `Authorization` header is there, is valid, contains the `Bearer` prefix, and extracts the JWT. If everything went well, it returns this string to the caller.

Back in the `authorize` function, the next step is to `decode` the JWT to get a valid `Claims` struct.

```rust
async fn authorize((role, headers): (Role, HeaderMap<HeaderValue>)) -> WebResult<String> {
    match jwt_from_header(&headers) {
        Ok(jwt) => {
            let decoded = decode::<Claims>(
                &jwt,
                &DecodingKey::from_secret(JWT_SECRET),
                &Validation::new(Algorithm::HS512),
            )
            .map_err(|_| reject::custom(Error::JWTTokenError))?;

            if role == Role::Admin && Role::from_str(&decoded.claims.role) != Role::Admin {
                return Err(reject::custom(Error::NoPermissionError));
            }

            Ok(decoded.claims.sub)
        }
        Err(e) => return Err(reject::custom(e)),
    }
}
```

If the JWT is expired, malformed, or in any way invalid, this decode step will fail and we will stop here. The `jsonwebtoken` library even gives us some customization options for the validation step, which is described well in the [official documentation](https://docs.rs/jsonwebtoken/7.2.0/jsonwebtoken/struct.Validation.html).

If the validation works, we can check the user role. If we’re in an `Admin` endpoint, the JWT role also needs to be `Admin`. If it isn’t, we throw a `NoPermissionError`.

Since we only have these two roles, this check is rather easy, but with several ore roles, it can get quite complex. A helpful library for handling such access control in a secure and maintainable way is [casbin](https://github.com/casbin/casbin-rs), which also has a well-maintained Rust crate.

Once the user passes the role check, we pass the user’s ID in the decorated handler. This is useful since the user’s identity will be relevant for many personalized endpoints, such as fetching a user profile or personal data.

This finishes the `with_auth` filter and we only have to use it for our handlers back in `main`.

```rust
async fn main() {
    ...
    let user_route = warp::path!("user")
        .and(with_auth(Role::User))
        .and_then(user_handler);
    let admin_route = warp::path!("admin")
        .and(with_auth(Role::Admin))
        .and_then(admin_handler);
    ...
}

pub async fn user_handler(uid: String) -> WebResult<impl Reply> {
    Ok(format!("Hello User {}", uid))
}

pub async fn admin_handler(uid: String) -> WebResult<impl Reply> {
    Ok(format!("Hello Admin {}", uid))
}
```

That was easy! Just decorate the existing handlers with the filter and put the incoming user ID in the handler signature. We also printed this user ID so we can test it later.

## Error handling

Good error handling is crucial when it comes to security. You don’t want to have a catch-all handler that leaks too much information to the outside. Errors should be helpful for the caller without revealing anything about the inner workings of the system.

In the `error.rs` module, we first define a custom `Error` type, an `ErrorResponse` type, and implement warp’s `Reject` trait so these errors can be used to return from handlers.

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("wrong credentials")]
    WrongCredentialsError,
    #[error("jwt token not valid")]
    JWTTokenError,
    #[error("jwt token creation error")]
    JWTTokenCreationError,
    #[error("no auth header")]
    NoAuthHeaderError,
    #[error("invalid auth header")]
    InvalidAuthHeaderError,
    #[error("no permission")]
    NoPermissionError,
}

#[derive(Serialize, Debug)]
struct ErrorResponse {
    message: String,
    status: String,
}

impl warp::reject::Reject for Error {}
```

Finally, we add the `handle_rejection` function, which was used initially in `main`.

```rust
pub async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
    let (code, message) = if err.is_not_found() {
        (StatusCode::NOT_FOUND, "Not Found".to_string())
    } else if let Some(e) = err.find::<Error>() {
        match e {
            Error::WrongCredentialsError => (StatusCode::FORBIDDEN, e.to_string()),
            Error::NoPermissionError => (StatusCode::UNAUTHORIZED, e.to_string()),
            Error::JWTTokenError => (StatusCode::UNAUTHORIZED, e.to_string()),
            Error::JWTTokenCreationError => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Internal Server Error".to_string(),
            ),
            _ => (StatusCode::BAD_REQUEST, e.to_string()),
        }
    } else if err.find::<warp::reject::MethodNotAllowed>().is_some() {
        (
            StatusCode::METHOD_NOT_ALLOWED,
            "Method Not Allowed".to_string(),
        )
    } else {
        eprintln!("unhandled error: {:?}", err);
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            "Internal Server Error".to_string(),
        )
    };

    let json = warp::reply::json(&ErrorResponse {
        status: code.to_string(),
        message,
    });

    Ok(warp::reply::with_status(json, code))
}
```

Most of this is boilerplate for dealing with rejections in warp and converting them to a JSON response at the end.

The interesting part is when we deal with our custom `Error` type. In this case, we map the errors, which can happen to status codes. Since we defined our error’s `Display` implementation to only contain a helpful error message, we can simply stringify the error.

If you add internal context to your errors, you should be very careful here and always define new, lightweight, and limited errors for exposing security-related errors to the outside. You never want to leak any information about inner workings, such as a stack trace.

It might also make sense, in a real system, to define an extra `SecurityError` type, which is carefully crafted to contain no sensible information and maps perfectly onto every possible auth-related case.

## Testing

Now that the authentication and authorization mechanism are both implemented, the last step is to see if it works.

We can start the server using `cargo run`, which will start a web server locally on port 8000.

Then, we can log in as a `User` and try to access the two endpoints:

```bash
curl http://localhost:8000/login -d '{"email": "user@userland.com", "pw": "1234"}' -H 'Content-Type: application/json'

{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIxIiwicm9sZSI6IlVzZXIiLCJleHAiOjE2MDMxMzQwODl9.dWnt5vfcGdwypEQUr3bLMrZYfdyxj3v6-io6VREWHXebMUCKBddf9xGcz4vHrCXruzx42zrS3Kygiqw3xV8W-A"}

curl http://localhost:8000/user -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIxIiwicm9sZSI6IlVzZXIiLCJleHAiOjE2MDMxMzQwODl9.dWnt5vfcGdwypEQUr3bLMrZYfdyxj3v6-io6VREWHXebMUCKBddf9xGcz4vHrCXruzx42zrS3Kygiqw3xV8W-A' -H 'Content-Type: application/json'

Hello User 1

curl http://localhost:8000/admin -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIxIiwicm9sZSI6IlVzZXIiLCJleHAiOjE2MDMxMzQwODl9.dWnt5vfcGdwypEQUr3bLMrZYfdyxj3v6-io6VREWHXebMUCKBddf9xGcz4vHrCXruzx42zrS3Kygiqw3xV8W-A' -H 'Content-Type: application/json'

{"message":"no permission","status":"401 Unauthorized"}
```

So far, so good. Logging in worked and returned a valid JWT. We used this JWT to make authenticated requests to `/user` and `/admin`. The first, as expected, worked and the second returned an error.

Let’s try the admin next:

```bash
curl http://localhost:8000/login -d '{"email": "admin@adminaty.com", "pw": "4321"}' -H 'Content-Type: application/json'

{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIyIiwicm9sZSI6IkFkbWluIiwiZXhwIjoxNjAzMTM0MjA1fQ.uYglVKRvb3h0bDC0Uz8FwGTu4v__Rl3toVI9fMI4_IT8keKde_SZRFQ4ii_PKzI4wjmDsZlnpULe6Tg0vWfEnw"}

curl http://localhost:8000/admin -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIyIiwicm9sZSI6IkFkbWluIiwiZXhwIjoxNjAzMTM0MjA1fQ.uYglVKRvb3h0bDC0Uz8FwGTu4v__Rl3toVI9fMI4_IT8keKde_SZRFQ4ii_PKzI4wjmDsZlnpULe6Tg0vWfEnw' -H 'Content-Type: application/json'

Hello Admin 2

curl http://localhost:8000/user -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJzdWIiOiIyIiwicm9sZSI6IkFkbWluIiwiZXhwIjoxNjAzMTM0MjA1fQ.uYglVKRvb3h0bDC0Uz8FwGTu4v__Rl3toVI9fMI4_IT8keKde_SZRFQ4ii_PKzI4wjmDsZlnpULe6Tg0vWfEnw' -H 'Content-Type: application/json'

Hello User 2
```

Great! The admin can access both endpoints and we logged the correct user ID. If this were a real system, we would write an exhaustive suite of tests for the validation, success, and error cases.

Fuzzing the auth-related endpoints is also a good way to increase the robustness of an implementation. Nothing ensures there are no weird edge cases left than sending billions of random values into something!

You can find the full example code on [GitHub](https://github.com/zupzup/rust-jwt-example).

## Conclusion

In this tutorial, we implemented a basic authentication and authorization model using JSON Web Tokens.

The `jsonwebtoken` crate is a mature and widely used option within the Rust ecosystem. While we used warp for this example, the ideas and techniques used here will translate very well to any other Rust web framework.

JWTs are a powerful tool for dealing with authorization and efficiently distributing information securely, and the Rust community proved up to the task once again — a great sign for it’s rising maturity in the area of web services.





