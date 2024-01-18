*This post was originally posted on the [LogRocket](https://blog.logrocket.com/parsing-in-rust-with-nom/) blog on 09.12.2020 and was cross-posted here by the author.*

In this tutorial, we’ll demonstrate how to write a very basic URL parser in Rust using the nom parser combinator library. 

## What are parser combinators?

Parser combinators are higher-order functions that can accept several parsers as input and return a new parser as its output.

This approach enables you to build parsers for simple tasks, such as parsing a certain string or a number, and compose them using combinator functions into a whole recursive descent parser.

Benefits of combinatory parsing include testability, maintainability, and readability; each moving part is rather small and self-isolated, making the whole parser a composition of modular components.

If you’re new to the concept, I highly recommend reading Bodil Stokke’s excellent [tutorial on parser combinators in Rust](https://bodil.lol/parser-combinators/).

## How does `nom` work?

[nom](https://github.com/Geal/nom) is a parser combinator library written in Rust that enables you to create safe parsers without hogging memory or compromising performance. It relies on Rust’s strong typing and memory safety to produce parsers that are both correct and performant, as well as functions, macros, and traits to abstract the error-prone plumbing.

To show how nom works, we’ll create a basic URL parser. We won’t implement the whole [URL spec](https://url.spec.whatwg.org/); that would far exceed the scope of this code example. Instead, we’ll take a few shortcuts.

The ultimate goal is to be able to parse valid URLs, such as `https://www.zupzup.org/about/?someVal=5&anotherVal=hello#anchor` and `http://user:pw@127.0.0.1:8080`, into a coherent structure, returning a somewhat usable error for invalid URLs in the process.

And since testability is touted as one of the benefits of parser combinators, we’ll write tests for most of the components to see that advantage in action.

Let’s get started!

## Setting up `nom`

To follow along, all you need is a recent Rust installation (1.44+).

First, create a new Rust project:

```bash
cargo new --lib rust-nom-example
cd rust-nom-example
```

Next, edit the `Cargo.toml` file and add the dependencies you’ll need:

```toml
[dependencies]
nom = "6.0"
```

Yup, all we need is the `nom` library in the latest version (6.0 at the time of writing).

## Data types

When writing a parser, it often makes sense to define the output structure first to get a feel for which parts you’ll need and what they look like.

In this case, we’re parsing a URL, so let’s define a structure for it:

```rust
#[derive(Debug, PartialEq, Eq)]
pub struct URI<'a> {
    scheme: Scheme,
    authority: Option<Authority<'a>>,
    host: Host,
    port: Option<u16>,
    path: Option<Vec<&'a str>>,
    query: Option<QueryParams<'a>>,
    fragment: Option<&'a str>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum Scheme {
    HTTP,
    HTTPS,
}

pub type Authority<'a> = (&'a str, Option<&'a str>);

#[derive(Debug, PartialEq, Eq)]
pub enum Host {
    HOST(String),
    IP([u8; 4]),
}

pub type QueryParam<'a> = (&'a str, &'a str);
pub type QueryParams<'a> = Vec<QueryParam<'a>>;
```

Let’s go through this line by line.

The fields are arranged according to the order in which they occur in a regular URI. First, we have the scheme. In this case, we’ll limit ourselves to `http://` and `https://`, but note that there are many other schemes available.

Next comes the `authority` part, which consists of a username and an optional password and is, in general, fully optional.

The host can be either an IP (in our case, only an IPv4), or a host string, such as `example.org`, followed by an optional port, which is just a number, such as `localhost:8080`.

After the port, we have the path, which is a sequence of `/-delimited strings, such as `/some/important/path`. This is optional as well, same as the query and fragment parts, which denote the `?query=some-value&another=5` and `#anchor` parts of a URL. The query is an optional list of string tuples and the fragment is simply an optional string.

If you’re confused by the lifetimes (`'a`) used throughout these types, don’t worry too much about them; they won’t really affect how we write the code. Essentially, instead of allocating a new string for each part of the URL, we can use pointers to parts of the input string and, as long as that input lives as long as our URI struct, we’re fine.

Before we start parsing, let’s define a helper for converting a valid scheme to our `Scheme` enum:

```rust
impl From<&str> for Scheme {
    fn from(i: &str) -> Self {
        match i.to_lowercase().as_str() {
            "http://" => Scheme::HTTP,
            "https://" => Scheme::HTTPS,
            _ => unimplemented!("no other schemes supported"),
        }
    }
}
```

With that out of the way, let’s take it from the top and start parsing the `scheme`.

## Error handling in `nom`

Before we start, let’s talk about error handling in `nom`. While we’re not going to go all the way here, we’ll try to at least give the caller a general idea of what went wrong at which parsing step.

For our purpose, we’ll use nom’s `context` combinator. In `nom`, a parser generally returns the following type:

```rust
type IResult<I, O, E = (I, ErrorKind)> = Result<(I, O), Err<E>>;
```

In our case, we have a result type that returns a tuple of the input value (`&str` — the input string), which is the remaining string to parse, and an output value. It also returns an error value in case the parsing fails.

The standard `IResult` only allows us to use Nom’s built-in error types, but what if we want to create custom errors or add some context to these errors?

The `ParseError` trait and `VerboseError` type enable us to build our own errors and add context to existing errors. For this simple example, we’ll add some context to our parsing errors. To make this convenient, let’s define our own result type:

```rust
type Res<T, U> = IResult<T, U, VerboseError<T>>;
```

This is essentially the same, except that it takes a `VerboseError`. This means we can use Nom’s `context` combinator, which allows us to implicitly add error context to any parser.

[nom‘s official docs](https://docs.rs/nom/6.0.0/nom/error/index.html) cover these options, but error handling is not the most intuitive thing.

To see it in action, let’s create our first parser for the scheme.

## Writing a parser in Rust

To parse the URL scheme, we want to match on either `http://`, or `https://`, nothing else. And since we’re using a powerful parser combinator library, we don’t need to write low-level parsers like that by hand. nom has us covered.

This [list of macros parsers and combinators](https://github.com/Geal/nom/blob/master/doc/choosing_a_combinator.md) has a great overview of what to use in certain use cases.

We’ll use the `tag_no_case` parser and the `alt` combinator to basically say, “Either lowercase (input) should be http://, or https://.” For this tutorial, we’ll just use normal functions, but do note that many of the parsers and combinators in nom are also available as macros.

Here’s how it looks in Rust using `nom`:

```rust
fn scheme(input: &str) -> Res<&str, Scheme> {
    context(
        "scheme",
        alt((tag_no_case("HTTP://"), tag_no_case("HTTPS://"))),
    )(input)
    .map(|(next_input, res)| (next_input, res.into()))
}
```

As you can see, we use the `context` combinator to wrap our actual parser and add the `scheme` context to it, so any errors triggered here will be marked with `scheme` in the result.

Once the whole parser is assembled using parsers and combinators, it’s called with the `input` string, which is our only input parameter. Then we `map` the result - which, as mentioned above, consists of the remaining input and the parsing output - to a different form, converting our parsed scheme into a Scheme enum with the `.into()` trait implementation mentioned earlier.

Let’s see if it works by writing a few tests:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use nom::{
        error::{ErrorKind, VerboseError, VerboseErrorKind},
        Err as NomErr,
    };

    #[test]
    fn test_scheme() {
        assert_eq!(scheme("https://yay"), Ok(("yay", Scheme::HTTPS)));
        assert_eq!(scheme("http://yay"), Ok(("yay", Scheme::HTTP)));
        assert_eq!(
            scheme("bla://yay"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    ("bla://yay", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    ("bla://yay", VerboseErrorKind::Nom(ErrorKind::Alt)),
                    ("bla://yay", VerboseErrorKind::Context("scheme")),
                ]
            }))
        );
    }
}
```

As you can see, in the success case, we get back our parsed `Scheme` enum and the remaining string to be parsed (`"yay"`). Also, if there is an error, we get a list of the different errors that were triggered and the context we defined (`"scheme"`).

In this case, both `tag` calls failed and, because of that, the `alt` combinator failed as well since it couldn’t produce a single value.

That wasn’t too hard. Then again, we basically just parsed a constant string. Let’s try something more advanced by trying to parse the `authority` part.

## Parsing URLs with authority

If we remember our URI struct from before, and especially the authority part, we’ll see that we’re searching for a fully optional structure. If it exists, there needs to be a username and an optional password.

This was the type alias we used:

```rust
pub type Authority<'a> = (&'a str, Option<&'a str>);
```

How could we go about this? In a URL, it looks like this:

`https://username:password@example.org`

The `:password` is optional, but in any case, there will be an `@` at the end, so we can start by using the terminated parser. This gives us a string, which is `terminated` by another string.

Within the authority, we see the : as a delimiter. According to the handy [docs](https://github.com/Geal/nom/blob/master/doc/choosing_a_combinator.md), it looks like we could use the `separated_pair` combinator, which gives us two values separated by a certain string. But how do we deal with the actual text? There are several options, one of which is to use the `alphanumeric1` parser. This generates an alphanumeric string that has at least one character.

For simplicity, we won’t worry about which characters can be used in different parts of the URL. It’s not relevant to writing and structuring the parser and would just make everything longer and more inconvenient. For our purposes, we’ll assume that most parts of the URL can consist of alphanumerics and, in some cases, hyphens and dots — which is, of course, wrong, according to the [URL Standard](https://url.spec.whatwg.org/#url-code-points).

Let’s look at the assembled `authority` parser:

```rust
fn authority(input: &str) -> Res<&str, (&str, Option<&str>)> {
    context(
        "authority",
        terminated(
            separated_pair(alphanumeric1, opt(tag(":")), opt(alphanumeric1)),
            tag("@"),
        ),
    )(input)
}
```

Let’s see if it works by running a few tests:

```rust
    #[test]
    fn test_authority() {
        assert_eq!(
            authority("username:password@zupzup.org"),
            Ok(("zupzup.org", ("username", Some("password"))))
        );
        assert_eq!(
            authority("username@zupzup.org"),
            Ok(("zupzup.org", ("username", None)))
        );
        assert_eq!(
            authority("zupzup.org"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (".org", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    ("zupzup.org", VerboseErrorKind::Context("authority")),
                ]
            }))
        );
        assert_eq!(
            authority(":zupzup.org"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (
                        ":zupzup.org",
                        VerboseErrorKind::Nom(ErrorKind::AlphaNumeric)
                    ),
                    (":zupzup.org", VerboseErrorKind::Context("authority")),
                ]
            }))
        );
        assert_eq!(
            authority("username:passwordzupzup.org"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (".org", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    (
                        "username:passwordzupzup.org",
                        VerboseErrorKind::Context("authority")
                    ),
                ]
            }))
        );
        assert_eq!(
            authority("@zupzup.org"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (
                        "@zupzup.org",
                        VerboseErrorKind::Nom(ErrorKind::AlphaNumeric)
                    ),
                    ("@zupzup.org", VerboseErrorKind::Context("authority")),
                ]
            }))
        )
    }
```

Looking good! We have cases for both values being there, the password missing, the `@` missing, and several other error cases.

Let’s move on to the `host` part.

## Rust parsing: Host, IP and port

Since the host part can consist of either a host string or an IP, this step will be more complex. To make matters worse, we can also have an optional `:port` at the end.

To keep things simple, we’ll only support IPv4 IPs. We’ll start with the host. Let’s look at the implementation and go through it step by step:

```rust
fn host(input: &str) -> Res<&str, Host> {
    context(
        "host",
        alt((
            tuple((many1(terminated(alphanumerichyphen1, tag("."))), alpha1)),
            tuple((many_m_n(1, 1, alphanumerichyphen1), take(0 as usize))),
        )),
    )(input)
    .map(|(next_input, mut res)| {
        if !res.1.is_empty() {
            res.0.push(res.1);
        }
        (next_input, Host::HOST(res.0.join(".")))
    })
}
```

The first thing you’ll notice is that there are two options (`alt`). In both cases, there is a `tuple`, a chain of parsers, which all have to work.

In the first case, we want to have one or more (`many1`) alphanumeric strings, including a hyphen, terminated with a . and ending with a TLD (alpha1).

The `alphanumerichyphen1` parser looks like this:

```rust
fn alphanumerichyphen1<T>(i: T) -> Res<T, T>
where
    T: InputTakeAtPosition,
    <T as InputTakeAtPosition>::Item: AsChar,
{
    i.split_at_position1_complete(
        |item| {
            let char_item = item.as_char();
            !(char_item == '-') && !char_item.is_alphanum()
        },
        ErrorKind::AlphaNumeric,
    )
}
```

This is a bit complex, but it’s basically just a copied version of nom’s `alphanumeric1` parser, with the - added. I’m not sure if this is the best way to do it, but it works!

In any case, there is a second option in a host, which is the case of one single string, such as `localhost`.

Why are we representing this in such a complex way with this useless-seeming `many_m_n` with 1 and 1? The issue here is that, within the `alt` combinator, all options have to return the same type - which, in this case, is a tuple of a string vector and another string.

We also see this in the map function, where, if the second part of the tuple isn’t empty (the top-level domain), we add it to the first part of the tuple. At the end, we create a HOST enum, joining the string parts with a `.` and creating the original host string.

Let’s write some tests:

```rust
    #[test]
    fn test_host() {
        assert_eq!(
            host("localhost:8080"),
            Ok((":8080", Host::HOST("localhost".to_string())))
        );
        assert_eq!(
            host("example.org:8080"),
            Ok((":8080", Host::HOST("example.org".to_string())))
        );
        assert_eq!(
            host("some-subsite.example.org:8080"),
            Ok((":8080", Host::HOST("some-subsite.example.org".to_string())))
        );
        assert_eq!(
            host("example.123"),
            Ok((".123", Host::HOST("example".to_string())))
        );
        assert_eq!(
            host("$$$.com"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    ("$$$.com", VerboseErrorKind::Nom(ErrorKind::AlphaNumeric)),
                    ("$$$.com", VerboseErrorKind::Nom(ErrorKind::ManyMN)),
                    ("$$$.com", VerboseErrorKind::Nom(ErrorKind::Alt)),
                    ("$$$.com", VerboseErrorKind::Context("host")),
                ]
            }))
        );
        assert_eq!(
            host(".com"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (".com", VerboseErrorKind::Nom(ErrorKind::AlphaNumeric)),
                    (".com", VerboseErrorKind::Nom(ErrorKind::ManyMN)),
                    (".com", VerboseErrorKind::Nom(ErrorKind::Alt)),
                    (".com", VerboseErrorKind::Context("host")),
                ]
            }))
        );
    }
```

Let’s move on to the IP case. At first, we need to be able to parse each individual part of an IPv4 IP (e.g.: 127.0.0.1):

```rust
fn ip_num(input: &str) -> Res<&str, u8> {
    context("ip number", n_to_m_digits(1, 3))(input).and_then(|(next_input, result)| {
        match result.parse::<u8>() {
            Ok(n) => Ok((next_input, n)),
            Err(_) => Err(NomErr::Error(VerboseError { errors: vec![] })),
        }
    })
}

fn n_to_m_digits<'a>(n: usize, m: usize) -> impl FnMut(&'a str) -> Res<&str, String> {
    move |input| {
        many_m_n(n, m, one_of("0123456789"))(input)
            .map(|(next_input, result)| (next_input, result.into_iter().collect()))
    }
}
```

To get each individual number, we try to find one to three consecutive digits using our `n_to_m_digits` parser and convert them to a `u8`.

With this out of the way, we can look at how to parse a full IP to an array of `u8`:

```rust
fn ip(input: &str) -> Res<&str, Host> {
    context(
        "ip",
        tuple((count(terminated(ip_num, tag(".")), 3), ip_num)),
    )(input)
    .map(|(next_input, res)| {
        let mut result: [u8; 4] = [0, 0, 0, 0];
        res.0
            .into_iter()
            .enumerate()
            .for_each(|(i, v)| result[i] = v);
        result[3] = res.1;
        (next_input, Host::IP(result))
    })
}
```

In this case, we’re looking for exactly three `ip_num`, followed by a . and another `ip_num`. In the mapping function, we then concatenate these individual results to the `Host::IP` enum with an array of u8.

Again, we’ll write a few tests to make sure this works:

```rust
    #[test]
    fn test_ipv4() {
        assert_eq!(
            ip("192.168.0.1:8080"),
            Ok((":8080", Host::IP([192, 168, 0, 1])))
        );
        assert_eq!(ip("0.0.0.0:8080"), Ok((":8080", Host::IP([0, 0, 0, 0]))));
        assert_eq!(
            ip("1924.168.0.1:8080"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    ("4.168.0.1:8080", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    ("1924.168.0.1:8080", VerboseErrorKind::Nom(ErrorKind::Count)),
                    ("1924.168.0.1:8080", VerboseErrorKind::Context("ip")),
                ]
            }))
        );
        assert_eq!(
            ip("192.168.0000.144:8080"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    ("0.144:8080", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    (
                        "192.168.0000.144:8080",
                        VerboseErrorKind::Nom(ErrorKind::Count)
                    ),
                    ("192.168.0000.144:8080", VerboseErrorKind::Context("ip")),
                ]
            }))
        );
        assert_eq!(
            ip("192.168.0.1444:8080"),
            Ok(("4:8080", Host::IP([192, 168, 0, 144])))
        );
        assert_eq!(
            ip("192.168.0:8080"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    (":8080", VerboseErrorKind::Nom(ErrorKind::Tag)),
                    ("192.168.0:8080", VerboseErrorKind::Nom(ErrorKind::Count)),
                    ("192.168.0:8080", VerboseErrorKind::Context("ip")),
                ]
            }))
        );
        assert_eq!(
            ip("999.168.0.0:8080"),
            Err(NomErr::Error(VerboseError {
                errors: vec![
                    ("999.168.0.0:8080", VerboseErrorKind::Nom(ErrorKind::Count)),
                    ("999.168.0.0:8080", VerboseErrorKind::Context("ip")),
                ]
            }))
        );
    }
```

To put it all together, we need another parser that tries both an IP and a host and returns a `Host`:

```rust
fn ip_or_host(input: &str) -> Res<&str, Host> {
    context("ip or host", alt((ip, host)))(input)
}
```

## Parsing the `path` in Rust

The next step is to tackle is the `path`. Here, again, we’ll just assume that strings within the path can only contain alphanumeric strings with hyphens and dots, using the following helper to parse them:

```rust
fn url_code_points<T>(i: T) -> Res<T, T>
where
    T: InputTakeAtPosition,
    <T as InputTakeAtPosition>::Item: AsChar,
{
    i.split_at_position1_complete(
        |item| {
            let char_item = item.as_char();
            !(char_item == '-') && !char_item.is_alphanum() && !(char_item == '.')
            // ... actual ascii code points and url encoding...: https://infra.spec.whatwg.org/#ascii-code-point
        },
        ErrorKind::AlphaNumeric,
    )
}
```

To parse the path, we want `/`-delimited strings parsed to a string vector:

```rust
fn path(input: &str) -> Res<&str, Vec<&str>> {
    context(
        "path",
        tuple((
            tag("/"),
            many0(terminated(url_code_points, tag("/"))),
            opt(url_code_points),
        )),
    )(input)
    .map(|(next_input, res)| {
        let mut path: Vec<&str> = res.1.iter().map(|p| p.to_owned()).collect();
        if let Some(last) = res.2 {
            path.push(last);
        }
        (next_input, path)
    })
}
```

We’ll always start with a `/`. This is already a valid path, but we can also have an additional `0`, or more (`many0`) /-terminated strings, with a final, optional string (e.g., `index.php`).

In the mapper, we check if the third part of the tuple (the last part) is there and, if so, push it to the path vector.

Let’s write some tests for the path as well:

```rust
    #[test]
    fn test_path() {
        assert_eq!(path("/a/b/c?d"), Ok(("?d", vec!["a", "b", "c"])));
        assert_eq!(path("/a/b/c/?d"), Ok(("?d", vec!["a", "b", "c"])));
        assert_eq!(path("/a/b-c-d/c/?d"), Ok(("?d", vec!["a", "b-c-d", "c"])));
        assert_eq!(path("/a/1234/c/?d"), Ok(("?d", vec!["a", "1234", "c"])));
        assert_eq!(
            path("/a/1234/c.txt?d"),
            Ok(("?d", vec!["a", "1234", "c.txt"]))
        );
    }
```

Looks good! We get the different parts of the path and the remaining string and everything is in a useful vector.

Let’s finish strong by parsing the query and the fragment parts of the URL.

## Query and fragment

The path is basically a key-value pair: the first one comes after a `?` and the rest are delimited with `&`. Again, we’ll restrict ourselves to the limited `url_code_points`.

```rust
fn query_params(input: &str) -> Res<&str, QueryParams> {
    context(
        "query params",
        tuple((
            tag("?"),
            url_code_points,
            tag("="),
            url_code_points,
            many0(tuple((
                tag("&"),
                url_code_points,
                tag("="),
                url_code_points,
            ))),
        )),
    )(input)
    .map(|(next_input, res)| {
        let mut qps = Vec::new();
        qps.push((res.1, res.3));
        for qp in res.4 {
            qps.push((qp.1, qp.3));
        }
        (next_input, qps)
    })
}
```

This is actually quite nice because the parser is intuitive and readable. We’re parsing a tuple of the first key-value pair after the `?`, separated with a `=` and then `0` or more of the same, starting with `&` instead of `?`.

Then, in the mapper, we simply put all of the key-value pairs into a vector and have our structure defined in the beginning:

```rust
pub type QueryParam<'a> = (&'a str, &'a str);
pub type QueryParams<'a> = Vec<QueryParam<'a>>;
```

Here’s a couple of basic tests:

```rust
    #[test]
    fn test_query_params() {
        assert_eq!(
            query_params("?bla=5&blub=val#yay"),
            Ok(("#yay", vec![("bla", "5"), ("blub", "val")]))
        );

        assert_eq!(
            query_params("?bla-blub=arr-arr#yay"),
            Ok(("#yay", vec![("bla-blub", "arr-arr"),]))
        );
    }
```

The last part is the fragment, which is just a `#` followed by a string:

```rust
fn fragment(input: &str) -> Res<&str, &str> {
    context("fragment", tuple((tag("#"), url_code_points)))(input)
        .map(|(next_input, res)| (next_input, res.1))
}
```

After all these the complex parsers we’ve covered, this is child’s play. For good measure, let’s write some sanity-check tests:

```rust
    #[test]
    fn test_fragment() {
        assert_eq!(fragment("#bla"), Ok(("", "bla")));
        assert_eq!(fragment("#bla-blub"), Ok(("", "bla-blub")));
    }
```

## Parsing in Rust with `nom`: The final test

Let’s put it all together into a top-level `uri` parser function that uses all the parts we defined above:

```rust
pub fn uri(input: &str) -> Res<&str, URI> {
    context(
        "uri",
        tuple((
            scheme,
            opt(authority),
            ip_or_host,
            opt(port),
            opt(path),
            opt(query_params),
            opt(fragment),
        )),
    )(input)
    .map(|(next_input, res)| {
        let (scheme, authority, host, port, path, query, fragment) = res;
        (
            next_input,
            URI {
                scheme,
                authority,
                host,
                port,
                path,
                query,
                fragment,
            },
        )
    })
}
```

We have a mandatory scheme, followed by an optional `authority`, followed by a mandatory `ip or host`. Then we have optional `port`, `path`, `query parameters`, and a `fragment`.

In the mapper, the only thing left is to create our `URI` struct from the parsed elements.

This is the point where you can see how nice and modular this whole structure is. If the uri function is your starting point, you can just go from top to bottom looking at each individual parser to understand what the whole thing is doing.

Of course, we’ll need some tests for `uri` as well:

```rust
    #[test]
    fn test_uri() {
        assert_eq!(
            uri("https://www.zupzup.org/about/"),
            Ok((
                "",
                URI {
                    scheme: Scheme::HTTPS,
                    authority: None,
                    host: Host::HOST("www.zupzup.org".to_string()),
                    port: None,
                    path: Some(vec!["about"]),
                    query: None,
                    fragment: None
                }
            ))
        );

        assert_eq!(
            uri("http://localhost"),
            Ok((
                "",
                URI {
                    scheme: Scheme::HTTP,
                    authority: None,
                    host: Host::HOST("localhost".to_string()),
                    port: None,
                    path: None,
                    query: None,
                    fragment: None
                }
            ))
        );

        assert_eq!(
            uri("https://www.zupzup.org:443/about/?someVal=5#anchor"),
            Ok((
                "",
                URI {
                    scheme: Scheme::HTTPS,
                    authority: None,
                    host: Host::HOST("www.zupzup.org".to_string()),
                    port: Some(443),
                    path: Some(vec!["about"]),
                    query: Some(vec![("someVal", "5")]),
                    fragment: Some("anchor")
                }
            ))
        );

        assert_eq!(
            uri("http://user:pw@127.0.0.1:8080"),
            Ok((
                "",
                URI {
                    scheme: Scheme::HTTP,
                    authority: Some(("user", Some("pw"))),
                    host: Host::IP([127, 0, 0, 1]),
                    port: Some(8080),
                    path: None,
                    query: None,
                    fragment: None
                }
            ))
        );
    }
```

It works! You can find the full example code at [GitHub](https://github.com/zupzup/rust-nom-parsing).

## Conclusion

What a ride! I hope this article got you excited about parsers and especially parser combinators in Rust.

The `nom` library, besides being ridiculously fast and foundational to many production-grade libraries and systems, provides a great API and some really good docs.

The Rust ecosystem offers more options for parsing, such as the [combine](https://github.com/Marwes/combine) library, which also implements parser combinators, and [pest](https://github.com/pest-parser/pest).

