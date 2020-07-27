In a [previous post](https://zupzup.org/casbin-http-role-auth/), we looked at how to use casbin to implement role-based authentication in a Go Web Service. In this post, we will do the same for a Rust Web Service.

In this example, we will define two user roles `member` and `admin`, create a user for each and use a [casbin](https://github.com/casbin/casbin-rs)-based authentication filter to handle authorization.

The authentication mechanism is, for simplicity, just based on the name of the user, using a hard-coded map of users.

For writing the web service providing this mechanism, we'll use [warp](https://github.com/seanmonstar/warp) here, but the basic concepts behind it should work with any web framework, with some of the implementation details changing to varying degrees.

Let's start!

*Disclaimer: Please don’t use the following example code as a template for a production-grade application, the code focuses on clarity, not on security.*

## Example

We'll start by defining some types. Since this is a simple example, we'll save our users and sessions in-memory:

```rust
mod handler;

const BEARER_PREFIX: &str = "Bearer ";

const MODEL_PATH: &str = "./auth/auth_model.conf";
const POLICY_PATH: &str = "./auth/policy.csv";

type UserMap = Arc<RwLock<HashMap<String, User>>>;
type Sessions = Arc<RwLock<HashMap<String, String>>>;
type WebResult<T> = std::result::Result<T, Rejection>;
type Result<T> = std::result::Result<T, Error>;
type SharedEnforcer = Arc<Enforcer>;

#[derive(Clone)]
pub struct UserCtx {
    pub user_id: String,
    pub token: String,
}

#[derive(Clone)]
pub struct User {
    pub user_id: String,
    pub name: String,
    pub role: String,
}

#[derive(Error, Debug)]
pub enum Error {
    #[error("error")]
    SomeError(),
    #[error("no authorization header found")]
    NoAuthHeaderFoundError,
    #[error("wrong authorization header format")]
    InvalidAuthHeaderFormatError,
    #[error("no user found for this token")]
    InvalidTokenError,
    #[error("error during authorization")]
    AuthorizationError,
    #[error("user is not unauthorized")]
    UnauthorizedError,
    #[error("no user found with this name")]
    UserNotFoundError,
}

impl warp::reject::Reject for Error {}
```

Alright - that's a lot of types. We start by defining the paths to the `casbin` model and policy files, followed by type aliases for our in-memory stores for users and sessions. The `UserMap` is a mapping of `user_id` to `User` structs and the `Sessions` map is a mapping of access tokens to user ids.

We also define some `Result` aliases, so we can differentiate between a result from a web-handler and an internal one. After the type aliases, we define the structs for `User` and `UserCtx`. This user context is what we'll pass through to authenticated handlers, so the handlers have access to the user_id and the authentication token - we'll see later on why this is useful.

Lastly, we define a custom `Error` type which implements warp's `Reject` trait, so we can return nice errors to the users.

Next, let's look at the `main` function to see a rough overview of the structure of the application:

```rust
#[tokio::main]
async fn main() {
    let user_map = Arc::new(RwLock::new(create_user_map()));
    let sessions: Sessions = Arc::new(RwLock::new(HashMap::new()));
    let enforcer = Arc::new(
        Enforcer::new(MODEL_PATH, POLICY_PATH)
            .await
            .expect("can read casbin model and policy files"),
    );

    let member_route = warp::path!("member")
        .and(with_auth(
            enforcer.clone(),
            user_map.clone(),
            sessions.clone(),
        ))
        .and_then(handler::member_handler);

    let admin_route = warp::path!("admin")
        .and(with_auth(
            enforcer.clone(),
            user_map.clone(),
            sessions.clone(),
        ))
        .and_then(handler::admin_handler);

    let login_route = warp::path!("login")
        .and(warp::post())
        .and(warp::body::json())
        .and(with_user_map(user_map.clone()))
        .and(with_sessions(sessions.clone()))
        .and_then(handler::login_handler);

    let logout_route = warp::path!("logout")
        .and(with_auth(
            enforcer.clone(),
            user_map.clone(),
            sessions.clone(),
        ))
        .and(with_sessions(sessions.clone()))
        .and_then(handler::logout_handler);

    let routes = member_route
        .or(admin_route)
        .or(login_route)
        .or(logout_route);

    warp::serve(routes).run(([0, 0, 0, 0], 8080)).await;
}
```

Let's step through this one-by-one. First, we initialize the above mentioned `sessions` and `user_map`. Then, importantly, we initialize the `casbin::Enforcer` using our model and policy files. This is the thing that enforces our defined authorization rules based on the request method, path and user id. We put this enforcer into an `Arc`, so we can pass references to it to other threads.

After that, we define the routes for `login`, `logout` and one route for members and admins respectively, to see if the authorization scheme works.

The model and policy files are defined as follows:

```bash
// model.conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && keyMatch(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

```bash
// policy.csv
p, admin, /*, *
p, anonymous, /login, *
p, member, /logout, *
p, member, /member, *
```

These definitions are quite simple. In the model, we define the different objects, the policy format and the matchers. In the policy, we actually provide concrete values for the `subject`, `object` and `action` - in our case these are the `user_role`, `path` and `request method`.

In warp, we use the `Filter` system to implement mechanisms for dealing with incoming requests, so let's create a new `Filter`, which handles authorization using `casbin`:

```rust
fn with_auth(
    enforcer: SharedEnforcer,
    user_map: UserMap,
    sessions: Sessions,
) -> impl Filter<Extract = (UserCtx,), Error = Rejection> + Clone {
    full()
        .and(headers_cloned())
        .and(method())
        .map(
            move |path: FullPath, headers: HeaderMap<HeaderValue>, method: Method| {
                (
                    path,
                    enforcer.clone(),
                    headers,
                    method,
                    user_map.clone(),
                    sessions.clone(),
                )
            },
        )
        .and_then(user_authentication)
}
```

The filter gets the full path using the `path::full()` filter, the headers and the request method. Then we pass these values, together with the enforcer, user map and sessions map to the `user_authentication` function, which looks like this:

```rust
async fn user_authentication(
    args: (
        FullPath,
        SharedEnforcer,
        HeaderMap<HeaderValue>,
        Method,
        UserMap,
        Sessions,
    ),
) -> WebResult<UserCtx> {
    let (path, enforcer, headers, method, user_map, sessions) = args;

    let token = token_from_header(&headers).map_err(|e| warp::reject::custom(e))?;
    let user_id = match sessions.read().await.get(&token) {
        Some(v) => v.clone(),
        None => return Err(warp::reject::custom(Error::InvalidTokenError)),
    };
    let user = match user_map.read().await.get(&user_id) {
        Some(v) => v.clone(),
        None => return Err(warp::reject::custom(Error::InvalidTokenError)),
    };
    match enforcer
        .enforce(&[&user.role.as_str(), &path.as_str(), &method.as_str()])
        .await
    {
        Ok(authorized) => {
            if authorized {
                Ok(UserCtx {
                    user_id: user.user_id,
                    token,
                })
            } else {
                Err(warp::reject::custom(Error::UnauthorizedError))
            }
        }
        Err(e) => {
            eprintln!("error during authorization: {}", e);
            Err(warp::reject::custom(Error::AuthorizationError))
        }
    }
}

fn token_from_header(headers: &HeaderMap<HeaderValue>) -> Result<String> {
    let header = match headers.get(AUTHORIZATION) {
        Some(v) => v,
        None => return Err(Error::NoAuthHeaderFoundError),
    };
    let auth_header = match from_utf8(header.as_bytes()) {
        Ok(v) => v,
        Err(_) => return Err(Error::NoAuthHeaderFoundError),
    };
    if !auth_header.starts_with(BEARER_PREFIX) {
        return Err(Error::InvalidAuthHeaderFormatError);
    }
    let without_prefix = auth_header.trim_start_matches(BEARER_PREFIX);
    Ok(without_prefix.to_owned())
}
```

Alright, so first we destructure the arguments. Then, we get the authorization token from the `Authorization` header, returning an error if it isn't there, or if it's invalid. Then we check if there is a session in existence with the token from the header.

If not, the user gets an error, if there is, we go one step further and see if the user the token belongs to still exists. If all of that worked out, we use the casbin `Enforcer`, using the path, request method and user role to see if the user has access to the requested resource. If any of these operations fail, an error is returned, otherwise the token and user id are forwarded to the handler function.

This is essentially the whole authorization magic and we can re-use this filter to secure endpoints as shown in the routes above.

To use the `user_map` and `sessions` map in handlers, we also need warp filters for those:

```rust
fn with_user_map(
    user_map: UserMap,
) -> impl Filter<Extract = (UserMap,), Error = Infallible> + Clone {
    warp::any().map(move || user_map.clone())
}

fn with_sessions(
    sessions: Sessions,
) -> impl Filter<Extract = (Sessions,), Error = Infallible> + Clone {
    warp::any().map(move || sessions.clone())
}
```

Alright. Let's create a hard-coded users map, so we can test the whole mechanism later on:

```rust
fn create_user_map() -> HashMap<String, User> {
    let mut map = HashMap::new();
    map.insert(
        String::from("21"),
        User {
            user_id: String::from("21"),
            name: String::from("herbert"),
            role: String::from("member"),
        },
    );
    map.insert(
        String::from("100"),
        User {
            user_id: String::from("100"),
            name: String::from("sibylle"),
            role: String::from("admin"),
        },
    );
    map.insert(
        String::from("1"),
        User {
            user_id: String::from("1"),
            name: String::from("gordon"),
            role: String::from("anonymous"),
        },
    );
    map
}
```

With that in place, let's look at the handler implementations next, starting with `login`.

```rust
#[derive(Deserialize, Debug)]
pub struct LoginRequest {
    pub name: String,
}

pub async fn login_handler(
    body: LoginRequest,
    user_map: UserMap,
    sessions: Sessions,
) -> WebResult<impl Reply> {
    let name = body.name;
    match user_map
        .read()
        .await
        .iter()
        .filter(|(_, v)| *v.name == name)
        .nth(0)
    {
        Some(v) => {
            let token = Uuid::new_v4().to_string();
            sessions
                .write()
                .await
                .insert(token.clone(), String::from(v.0));
            Ok(token)
        }
        None => Err(warp::reject::custom(Error::UserNotFoundError)),
    }
}
```

In order to log a user in, we search for the user in the `user_map`, create a new unique session token (a uuid in this case), and create a session mapping from that token to the user_id, returning the token to the user.

Logging out and the endpoints for checking if the user has access as a member, or admin, are rather simple:

```rust
pub async fn logout_handler(user_ctx: UserCtx, sessions: Sessions) -> WebResult<impl Reply> {
    sessions.write().await.remove(&user_ctx.token);
    Ok("success")
}

pub async fn member_handler(user_ctx: UserCtx) -> WebResult<impl Reply> {
    Ok(format!("Member with id {}", user_ctx.user_id))
}

pub async fn admin_handler(user_ctx: UserCtx) -> WebResult<impl Reply> {
    Ok(format!("Admin with id {}", user_ctx.user_id))
}
```

Logging out essentially just removes the session mapping. The test endpoints just return a string, so we can check if they worked. The whole mechanism for checking the authorization is handled on the `Filter` level mentioned above. This is nice, as it means inside the handlers, we don't have to deal with authorization at all.

That's it! After running the app using `cargo run`, we can test that it works using cURL:

```bash
curl -X POST http://localhost:8080/login -d '{ "name": "herbert" }' -H "content-type: application/json"
=> $TOKEN
curl http://localhost:8080/member -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 200
curl http://localhost:8080/admin -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 200

curl -X POST http://localhost:8080/login -d '{ "name": "sibylle" }' -H "content-type: application/json"
=> $TOKEN
curl http://localhost:8080/member -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 200
curl http://localhost:8080/admin -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 401

curl -X POST http://localhost:8080/login -d '{ "name": "gordon" }' -H "content-type: application/json"
=> $TOKEN
curl http://localhost:8080/member -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 401
curl http://localhost:8080/admin -H "authorization: Bearer $TOKEN" -H "content-type: application/json"
=> 401
```

First, we log in as admin, then check that both `/member` and `/admin` work. Then, logging in as a member, we make sure only `/member` works, but not `/admin`. And lastly, logging in as neither an admin, nor a member, we make sure that both `/admin` and `/member` don't work.

The full example code can be found [here](https://github.com/zupzup/rust-casbin-example)

## Conclusion

This post showed off the fantastic [casbin](https://github.com/casbin/casbin-rs) library for doing authorization in Rust web services. The nice thing about casbin is, that the general concept is supported by a huge amount of languages, with middlewares for many well-known web servers. This leads to wider adoption and in the process, more people looking at the code, which translates to better security.

The policy-based approach behind casbin is both simple, yet powerful and can be used to achieve many different authorization models.

#### Resources

* [Code Example](https://github.com/zupzup/rust-casbin-example)
* [Blog Post about casbin in Go](https://zupzup.org/casbin-http-role-auth/)
* [Casbin-Rs](https://github.com/casbin/casbin-rs)
