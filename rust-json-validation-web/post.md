*This post was originally posted on the [LogRocket](https://blog.logrocket.com/json-input-validation-in-rust-web-services/) blog on 14.10.2020 and was cross-posted here by the author.*

When building web services with somewhat complex domain objects, good input validation is paramount. Validation of user input is fundamentally important in terms of not just security — never trust external inputs - but also usability.

If the caller of a REST endpoint runs into an error, you want to be able to tell them what went wrong instead of simply displaying `400 Bad Request`. Optimal input error handling is done in such a way that if you get invalid JSON, you can tell the user where, roughly, the error is located in the request payload.

If the request payload is valid JSON, the next step is to make sure it adheres to your specification. Rust, with the fantastic [serde](https://blog.logrocket.com/json-and-rust-why-serde_json-is-the-top-choice/) crate, helps here because deserialization to a struct will fail if a wrong data type is used. Again, if this happens, you need to tell the user that, for instance, `name` needs to be a `String` and not an `Object`, instead of simply replying `JSON Parse Error`.

But we’re still not quite done. Once the incoming JSON has been properly validated and parsed to a struct, our own validation starts — the validation based on our business logic. For example, you might want to validate that an `email` is actually an email according to a specification, or that a `username` does not contain forbidden characters, is a certain length, and so on. This part is easier to handle than the others, but it often results in many small, error-prone validation functions, which are hard to combine and distill into nice, clear error messages.

In this tutorial, we’ll explain how to solve these issues in Rust within a [warp](https://github.com/seanmonstar/warp) web service. Our solutions will be intuitive and user-friendly without compromising the maintainability of the code.

Let’s get started!

## Setup

To follow along, all you need is a reasonably recent Rust installation (1.39+) and a tool to make HTTP requests, such as cURL.

First, create a new Rust project.

```bash
cargo new rust-json-validation-example
cd rust-json-validation-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need.

```toml
tokio = { version = "0.2", features = ["macros", "sync", "rt-threaded"] }
warp = "0.2"
serde = {version = "1.0", features = ["derive"] }
serde_json = "1.0"
validator = "0.10"
validator_derive = "0.10"
serde_path_to_error = "0.1"
thiserror = "1.0.20"
bytes = "0.5.6"
```

You’ll need warp and [tokio](https://github.com/tokio-rs/tokio) for the web server and [serde](https://github.com/serde-rs/serde) to deserialize the incoming body. The `serde_path_to_error` library will be your first stop to improve validation, and the `validator` and `validator_derive` crates will help later on with data validation.

## JSON body validation

To demonstrate the default behavior in warp when sending something that’s invalid JSON or can’t be successfully deserialized, let’s create a small web server with a single route.

```rust
type Result<T> = std::result::Result<T, Rejection>;

#[derive(Deserialize, Debug)]
struct CreateRequest {
    pub email: String,
    pub address: Address,
    pub pets: Vec<Pet>,
}

#[derive(Deserialize, Debug)]
struct Address {
    pub street: String,
    pub street_no: usize,
}

#[derive(Deserialize, Serialize, Debug)]
struct Pet {
    pub name: String,
}

#[tokio::main]
async fn main() {
    let basic = warp::path!("create-basic")
        .and(warp::post())
        .and(warp::body::json())
        .and_then(create_handler);

    let routes = basic
        .recover(handle_rejection);

    println!("Server started at localhost:8080!");
    warp::serve(routes).run(([127, 0, 0, 1], 8080)).await;
}

async fn create_handler(body: CreateRequest) -> Result<impl Reply> {
    Ok(format!("called with: {:?}", body))
}

#[derive(Serialize)]
struct ErrorResponse {
    message: String,
    errors: Option<Vec<FieldError>>,
}

#[derive(Serialize)]
struct FieldError {
    field: String,
    field_errors: Vec<String>,
}

pub async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
    let (code, message, errors) = if err.is_not_found() {
        (StatusCode::NOT_FOUND, "Not Found".to_string(), None)
    } else if let Some(e) = err.find::<warp::filters::body::BodyDeserializeError>() {
        (
            StatusCode::BAD_REQUEST,
            e.source()
                .map(|cause| cause.to_string())
                .unwrap_or_else(|| "BAD_REQUEST".to_string()),
            None,
        )
    } else {
        eprintln!("unhandled error: {:?}", err);
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            "Internal Server Error".to_string(),
            None,
        )
    };

    let json = warp::reply::json(&ErrorResponse {
        message: message.into(),
        errors,
    });
    Ok(warp::reply::with_status(json, code))
}
```

With this, we created our request object — in this case just some fields we can have some fun validating, such as `email`, an `address`, and a list of `Pets`.

Then, we created a minimal handler, which takes this as a JSON body and, when called, prints it.

Below that, we defined some basic error handling within the `handle_rejection` helper. This is warp’s way of dealing with errors, which, in warp’s terminology, are called `Rejections`. You can ignore the `FieldError` for now; we’ll revisit it later on when we do data validation.

There is also a predefined case for errors happening during body deserialization: `warp::filters::body::BodyDeserializeError`. If we encounter an error like that, we return a 400 error with the cause of the error converted to a string.

Here’s how that looks from the caller’s side. Let’s send a wrong payload with cURL:

```bash
curl -X POST http://localhost:8080/create-basic -H "Content-Type: application/json" -d '{ "email": 1, "address": { "street": "warpstreet", "street_no": 1 }, "pets": [{ "name": "nacho" }] }'
```

We get the following error.

```json
{"message":"invalid type: integer `1`, expected a string at line 1 column 13","errors":null}
```

It’s not too bad - at the very least, it tells us the line and column where the error was (email is a number in our wrong payload) — but it would be a lot nicer to have the error actually tell you that `email` was the problem.

In any case, this is much better than not having a `BodyDeserializeError` handler. iIn that case, the response would be:

```json
{"message":"Internal Server Error","errors":null}
```

## Better JSON validation

To improve the error, we’ll use the `serde_path_to_error` crate, which, as the name suggests, converts the serde error to the path in the JSON where it happened.

Let’s create another handler for it and see what changes.

```rust
#[tokio::main]
async fn main() {
...
    let basic_path = warp::path!("create-path")
        .and(warp::post())
        .and(warp::body::aggregate())
        .and_then(create_handler_path);

    let routes = basic
        .or(basic_path)
        .recover(handle_rejection);
...
}

async fn create_handler_path(buf: impl Buf) -> Result<impl Reply> {
    let des = &mut serde_json::Deserializer::from_reader(buf.reader());
    let body: CreateRequest = serde_path_to_error::deserialize(des)
        .map_err(|e| reject::custom(Error::JSONPathError(e.to_string())))?;
    Ok(format!("called with: {:?}", body))
}

#[derive(Error, Debug)]
enum Error {
    #[error("JSON path error: {0}")]
    JSONPathError(String),
}

impl warp::reject::Reject for Error {}

pub async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
...
    } else if let Some(e) = err.find::<Error>() {
        match e {
            Error::JSONPathError(_) => (StatusCode::BAD_REQUEST, e.to_string(), None),
        }
...
}
```

As you can see, we added another handler at the path `create-path`. The biggest difference here is that we used `warp::body::aggregate()` instead of `warp::body::json()`, which would’ve automatically deserialized the payload and errors before we had a chance to intervene.

In this case, we wanted the aggregated byte buffer of the payload to work with. In practice, if you were to reuse this pattern a lot, you would probably write your own custom `json()` filter.

In the handler, we created a deserializer and used the `serde_path_to_error::deserialize` function to deserialize the incoming JSON buffer into a `CreateRequest` struct. If this fails, it triggers a `JSONPathError` - a custom error type that implements warp’s `Reject` trait, which is defined below it. This is a simple way to perform custom error handling in warp.

Let’s see what the response looks like now. We’ll send the same payload as before.

```json
{"message":"JSON path error: email: invalid type: integer `1`, expected a string at line 1 column 13","errors":null}
```

What if there’s an error deeper down in the structure, such as the name of a pet?

```bash
curl -X POST http://localhost:8080/create-path -H "Content-Type: application/json" -d '{ "email": "mario@zupzup.org", "address": { "street": "warpstreet", "street_no": 1 }, "pets": [{ "name": 1 }] }'
```

This would give the following error.

```json
{"message":"JSON path error: pets[0].name: invalid type: integer `1`, expected a string at line 1 column 107","errors":null}
```

That’s a lot better. Now the error tells the caller which field is the problem and even offers a hint on how to fix it. In terms of handling deserialization errors, I don’t think we’ll be able to do much better.

One alternative approach would be to put the response into a data structure that would be easier to parse — for example, so a GUI could create a clean error message out of it. But since we’re talking about straight-up invalid requests, this isn’t as important in this case. It becomes more relevant when we get to the data validation stage.

## Data validation

Data validation, in this case, simply means making sure the provided data adheres to our specification of what values should look like — for example, that an `email` or `url` is valid, or that we only want to accept `https://` URLs.

To achieve this, you could simply write a validation function for each field and each struct that comes in, but that gets tedious. It’s also hard to combine in a larger app with many different, complex domain objects, leading to code and especially logic duplication that is error-prone.

Among other libraries in the Rust ecosystem, the validator crate helps tremendously here. This macro-based library enables you to declaratively define validation rules for structs and fields. This approach might sound familiar since there are many such libraries in other languages, such as marshmallow, Java Spring etc.

Let’s see how it works in practice:

```rust
use validator::{Validate, ValidationErrors, ValidationErrorsKind};

#[derive(Deserialize, Debug, Validate)]
struct CreateRequest {
    #[validate(email)]
    pub email: String,
    #[validate]
    pub address: Address,
    #[validate]
    pub pets: Vec<Pet>,
}

#[derive(Deserialize, Debug, Validate)]
struct Address {
    #[validate(length(min = 2, max = 10))]
    pub street: String,
    #[validate(range(min = 1))]
    pub street_no: usize,
}

#[derive(Deserialize, Serialize, Debug, Validate)]
struct Pet {
    #[validate(length(min = 3, max = 20))]
    pub name: String,
}

#[tokio::main]
async fn main() {
...
    let basic_path_validator = warp::path!("create-validator")
        .and(warp::post())
        .and(warp::body::aggregate())
        .and_then(create_handler_validator);

    let routes = basic
        .or(basic_path)
        .or(basic_path_validator)
        .recover(handle_rejection);
...
}

async fn create_handler_validator(buf: impl Buf) -> Result<impl Reply> {
    let des = &mut serde_json::Deserializer::from_reader(buf.reader());
    let body: CreateRequest = serde_path_to_error::deserialize(des)
        .map_err(|e| reject::custom(Error::JSONPathError(e.to_string())))?;

    body.validate()
        .map_err(|e| reject::custom(Error::ValidationError(e)))?;
    Ok(format!("called with: {:?}", body))
}

#[derive(Error, Debug)]
enum Error {
    #[error("JSON path error: {0}")]
    JSONPathError(String),
    #[error("validation error: {0}")]
    ValidationError(ValidationErrors),
}

...
```

We added another handler `create-validator`, which uses the same approach we used for `create-path`. So we established nice JSON input handling and then quite simply called `body.validate()` to validate the incoming data, propagating a new custom error with any errors.

The real sauce however, is in the definition of our data structs. There we derived the `Validate` trait for our structs and, for each field, used the `validate` macro to define rules regarding whether and how to validate this field. This could be fancy stuff such as email validation, basic stuff such as string length, or even custom validation functions. Check out the [documentation](https://github.com/Keats/validator) to see more examples; it’s quite powerful and customizable.

In the above example, the error handling for `ValidationError` was omitted because it’s a bit complicated. The `ValidationErrors` object we get back from the `validator` crate contains all the information we would want and can even be stringified to give a somewhat useful error. But in our case, we’d like to parse it into our custom `FieldError` structure where we can map errors to the fields in which they occurrred.

Let’s look at a basic way to achieve this.

```rust
pub async fn handle_rejection(err: Rejection) -> std::result::Result<impl Reply, Infallible> {
...
    } else if let Some(e) = err.find::<Error>() {
        match e {
            Error::JSONPathError(_) => (StatusCode::BAD_REQUEST, e.to_string(), None),
            Error::ValidationError(val_errs) => {
                let errors: Vec<FieldError> = val_errs
                    .errors()
                    .iter()
                    .map(|error_kind| FieldError {
                        field: error_kind.0.to_string(),
                        field_errors: match error_kind.1 {
                            ValidationErrorsKind::Struct(struct_err) => {
                                validation_errs_to_str_vec(struct_err)
                            }
                            ValidationErrorsKind::Field(field_errs) => field_errs
                                .iter()
                                .map(|fe| format!("{}: {:?}", fe.code, fe.params))
                                .collect(),
                            ValidationErrorsKind::List(vec_errs) => vec_errs
                                .iter()
                                .map(|ve| {
                                    format!(
                                        "{}: {:?}",
                                        ve.0,
                                        validation_errs_to_str_vec(ve.1).join(" | "),
                                    )
                                })
                                .collect(),
                        },
                    })
                    .collect();

                (
                    StatusCode::BAD_REQUEST,
                    "field errors".to_string(),
                    Some(errors),
                )
            }
        }
...
}

fn validation_errs_to_str_vec(ve: &ValidationErrors) -> Vec<String> {
    ve.field_errors()
        .iter()
        .map(|fe| {
            format!(
                "{}: errors: {}",
                fe.0,
                fe.1.iter()
                    .map(|ve| format!("{}: {:?}", ve.code, ve.params))
                    .collect::<Vec<String>>()
                    .join(", ")
            )
        })
        .collect()
}
```

As you can see, this is a bit more complex than you might have expected. Theoretically, for an arbitrarily deeply nested structure, you could arbitrarily deeply nest the resulting field errors structure as well. We need to make a decision and define that the cut-off point is `1` level. Beyond the `1` level of depth, we simply added the whole string of the underlying field errors into the topmost field error.

In practice, this could be solved nicely using recursion, but we’ll keep it simple here.

Let’s go through the code to see what’s going on here. As mentioned above, we get a `ValidationErrors` object from the library, which we want to parse into a `Vec<FieldError`. So far, so good.

If we iterate over the `ValidationErrors` using the `errors()` iterator, we get a collection of `ValidationErrorKinds`. There are three error types:

1. Field means the error happened in a field, such as `email` - easy
2. Struct means the error happened in a nested struct
3. List means the error happened in a nested list

For each error kind, we get a tuple of the field name and the actual error. The field name we can simply put into our `field` property on `FieldError`. Since we get a `&&str` and want a `String`, we need to call `to_string()`.

The field errors are a bit tougher. We need to match on the three above mentioned cases. If we encounter a `ValidationErrorsKind::Field`, it’s rather simple and we simply format the error into a string.

If we run into a `ValidationErrorsKind::Struct`, we call the `validation_errs_to_str_vec` helper, which iterates over all errors within this struct, formats them to Strings, and concatenates them with a `,`, returning a list of these concatenated errors within the struct.

For the last case, the `ValidationErrorsKind::List`, we do a combination of the two, since every item of the list could also itself be a struct again. wWe call the `validation_errs_to_str_vec` for each field, tagging it in the string with the list index at which the errors occurred.

If you had trouble following this last bit, don’t worry about it too much. In practice, the returned validation errors are parsed and passed back in a different way based on the circumstances. This is just one simplified and, admittedly, imperfect way of doing it.

That said, let’s see at what kind of errors we get if we violate some of the data validation rules.

First, let’s see what happens with an invalid email.

```bash
curl -X POST http://localhost:8080/create-validator -H "Content-Type: application/json" -d '{ "email": "chipexample.com", "address": { "street": "warpstreet", "street_no": 1 }, "pets": [{ "name": "nacho" }] }'
```

=>

```json
{"message":"field errors","errors":[{"field":"email","field_errors":["email: {\"value\": String(\"chipexample.com\")}"]}]}
```

We get an array of errors, the field is set correctly, and, within the field, the field errors also contain the invalid email error showing the wrong value.

Now let’s add another error to this and use a street name that’s too long for our rules.

```bash
curl -X POST http://localhost:8080/create-validator -H "Content-Type: application/json" -d '{ "email": "chipexample.com", "address": { "street": "warpstreetistoolonghere", "street_no": 1 }, "pets": [{ "name": "nacho" }] }'
```

=>

```json
{"message":"field errors","errors":[{"field":"address","field_errors":["street: errors: length: {\"max\": Number(10), \"min\": Number(2), \"value\": String(\"warpstreetistoolonghere\")}"]},{"field":"email","field_errors":["email: {\"value\": String(\"chipexample.com\")}"]}]}%
```

The second error is correctly added and we get some information about the length, which is expected here. As you can see in the formatted internal error string, we could go a lot further here and create a nice, structured error, giving the user fields for the `max` and `min`.

Finally, let’s try the nested case and send a pet name that is too short.

```bash
curl -X POST http://localhost:8080/create-validator -H "Content-Type: application/json" -d '{ "email": "chip@example.com", "address": { "street": "warpstreet", "street_no": 1 }, "pets": [{ "name": "" }, {"name": "a"}] }'
```

=>

```json
{"message":"field errors","errors":[{"field":"pets","field_errors":["0: \"name: errors: length: {\\\"value\\\": String(\\\"\\\"), \\\"min\\\": Number(3), \\\"max\\\": Number(20)}\"","1: \"name: errors: length: {\\\"value\\\": String(\\\"a\\\"), \\\"max\\\": Number(20), \\\"min\\\": Number(3)}\""]}]}
```

Since these are errors within a list, and since we implemented this in a very simple way, the errors are just concatenated inside and in the `field_errors` array, we simply get the error strings for the index `0` and `1`, showing that the names were too short.

You can find the full code for this tutorial on [GitHub](https://github.com/zupzup/example-rust-json-input-validation).

## Conclusion

JSON input validation is a core concern in any modern web application, and the Rust ecosystem already has some great tools for dealing with it.

In this example, we looked at how to deal with JSON deserialization errors and validate the data in the next step. There are many ways to do this — we just went over one of the myriad approaches to showing off two specific libraries in the ecosystem — but I hope it helped you get a feeling for how you can tackle similar problems in Rust.

