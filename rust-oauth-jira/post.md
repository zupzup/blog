I recently was in the situation that I had to implement a connection to the JIRA API, which means the privilege of having to deal with their OAuth 1.0a implementation.

Unfortunately, JIRA at this point in time uses `RSA-SHA1` in their OAuth 1.0a scheme, which I wasn't able to find a crate for I was happy with (the OAuth client part, not the crypto). So I decided to write it myself and this post is a description of some parts that came out of that edneavour.

The following example should not be considered as a full-fledged or production-ready OAuth-Client implementation based on the official spec by any means. What I implemented is loosely based on [node-oauth](https://github.com/ciaranj/node-oauth) as that was what I was trying to replace in this case.

The whole process of the authentication dance consists of the following steps:

* Get a Request Token
* Send the user to the Verification URL using the request token
* Coming back from the Verification URL, use the token and verification token to
* Get an Access Token

In the next section, we'll look at how a very basic implementation of these steps could look like in rust.

## Implementation

First up, we define some constants.

```rust
const OAUTH_HEADER_PREFIX: &str = "OAuth ";
const OAUTH_CONSUMER_KEY: &str = "oauth_consumer_key";
const OAUTH_NONCE: &str = "oauth_nonce";
const OAUTH_SIGNATURE_METHOD: &str = "oauth_signature_method";
const OAUTH_TIMESTAMP: &str = "oauth_timestamp";
const OAUTH_SIGNATURE: &str = "oauth_signature";
const OAUTH_TOKEN: &str = "oauth_token";
const OAUTH_VERSION: &str = "oauth_version";
const OAUTH_CALLBACK: &str = "oauth_callback";
const OAUTH_VERIFIER: &str = "oauth_verifier";
const OAUTH_1: &str = "1.0";

const EQUALS: char = '=';
const COMMA: char = ',';
const QUOTE: char = '"';
const AMPERSAND: char = '&';

const SIG_METHOD: &str = "RSA-SHA1";
const CONSUMER_KEY: &str = "SomeKey";
```

Above, we defined the field names we'll need in the OAuth requests, as well as some concatenation characters, the signature method we'll use and our Consumer Key.

Next up are some utility functions we'll need.

```rust
fn get_timestamp() -> u64 {
    time::SystemTime::now()
        .duration_since(time::UNIX_EPOCH)
        .unwrap()
        .as_secs()
}

fn get_nonce() -> String {
    let nonce: String = thread_rng().sample_iter(Alphanumeric).take(8).collect();
    nonce
}

fn encode(input: &str) -> String {
    percent_encode(input.as_bytes(), FRAGMENT).to_string()
}
```

We use `percent_encode` to urlencode strings and we need functions for getting the current timestamp and a random nonce, which we generate using `rand`.

Next up, we'll look at how we create the `oauth_signature` needed to sign oauth 1 requests. In our case, because JIRA uses it, we use the `RSA-SHA1` algorithm.

```rust
fn sign(private_key: &[u8], input: &str) -> String {
    let key = PKey::private_key_from_pem(&private_key).unwrap();
    let mut signer = Signer::new(MessageDigest::sha1(), &key).unwrap();
    signer.update(input.as_bytes()).unwrap();
    let signature = signer.sign_to_vec().unwrap();
    base64::encode_block(&signature)
}
```

For the signing, we use the `openssl` crate, which provides bindings to openssl. We parse the given `private key` to a PKey, instantiate a `sha1` Signer and sign our input. Afterwards, the created signature is returned encoded in base64.

The most relevant part of this implementation is the logic to create the `OAuth Header`. This is a special `Authorization` header, which includes oauth metadata, the oauth tokens and the signature over these values.

In order to create this header, we first need to encode the oauth and request parameters:

```rust
let mut params = BTreeMap::new();
params.insert(OAUTH_CONSUMER_KEY.to_owned(), CONSUMER_KEY.to_owned());
params.insert(OAUTH_NONCE.to_owned(), nonce.clone());
params.insert(OAUTH_VERSION.to_owned(), OAUTH_1.to_owned());
params.insert(OAUTH_TIMESTAMP.to_owned(), timestamp.to_string());
params.insert(OAUTH_SIGNATURE_METHOD.to_owned(), SIG_METHOD.to_owned());

for (key, value) in query {
    params.insert(key.into_owned(), value.into_owned());
}

let parameters = create_params(&params);

let encoded_url = encode(&url_without);
let encoded_parameters = encode(&parameters);
let combined = format!("GET&{}&{}", encoded_url, encoded_parameters);

let b64 = sign(&private_key, &combined);

for (key, _) in query {
    params.remove(key.as_ref());
}

params.insert(OAUTH_SIGNATURE.to_owned(), b64);

let header = create_header_params(&params);
```

Above, we first create a `BTreeMap` for the parameters. We use a sorted map here, because based on the OAuth spec, the parameter pairs need to be sorted. Then, we add the basic oauth parameters to the map. This includes the `Consumer Key, Nonce, Version, Timestamp` and the `Signature Method`.

Then, for the signature we use the `create_params` function to create the encoded parameter string:

```rust
fn create_params(params: &BTreeMap<String, String>) -> String {
    let mut res = String::new();
    for (i, (key, value)) in params.iter().enumerate() {
        res.push_str(key);
        res.push(EQUALS);
        res.push_str(&encode(value));
        if i < params.len() - 1 {
            res.push(AMPERSAND);
        }
    }
    res
}
```

Here, we pass in the collected oauth parameters and put them in the form `key=value&otherkey=othervalue` with the values being url encoded. The resulting string, as well as the url are then again url encoded and combined with the stringified HTTP method. This string is then passed to the above mentioned `sign` function to create the signature.

If there are more query parameters, we add those as well and finally we add the signature at the end of the parameter list.

This parameter map is then passed to the `create_header_params` function:

```rust
fn create_header_params(params: &BTreeMap<String, String>) -> String {
    let mut header = String::from(OAUTH_HEADER_PREFIX);
    for (key, value) in params.iter() {
        if *key != OAUTH_SIGNATURE {
            header.push_str(key);
            header.push(EQUALS);
            header.push(QUOTE);
            header.push_str(&encode(value));
            header.push(QUOTE);
            header.push(COMMA);
        }
    }
    header.push_str(OAUTH_SIGNATURE);
    header.push(EQUALS);
    header.push(QUOTE);
    header.push_str(&encode(
        params
            .get(OAUTH_SIGNATURE)
            .expect("signature needs to be there"),
    ));
    header.push(QUOTE);
    header
}
```

The format of these parameters is `key="value,...,oauth_signature="..."`, so we build a string to produce that. And the resulting string is actually already the `Authorization` Header for requests.

Alright, with all of that out of the way, let's start with the first OAuth step - getting a request token:

```rust
async fn get_request_token(private_key: &[u8]) -> Result<()> {
    let nonce = get_nonce();
    let timestamp = get_timestamp();

    let callback_url = "http://localhost:3000/cb";
    let url = "https://some-jira.atlassian.net/plugins/servlet/oauth/request-token";

    let mut params = BTreeMap::new();
    params.insert(OAUTH_CONSUMER_KEY.to_owned(), CONSUMER_KEY.to_owned());
    params.insert(OAUTH_NONCE.to_owned(), nonce.clone());
    params.insert(OAUTH_VERSION.to_owned(), OAUTH_1.to_owned());
    params.insert(OAUTH_TIMESTAMP.to_owned(), timestamp.to_string());
    params.insert(OAUTH_SIGNATURE_METHOD.to_owned(), SIG_METHOD.to_owned());
    params.insert(OAUTH_CALLBACK.to_owned(), callback_url.to_owned());

    let encoded_url = encode(url);
    let parameters = create_params(&params);
    let encoded_parameters = encode(&parameters);

    let combined = format!("POST&{}&{}", encoded_url, encoded_parameters);

    let b64 = sign(&private_key, &combined);

    params.insert(OAUTH_SIGNATURE.to_owned(), b64);

    let header = create_header_params(&params);

    let res = make_request(Method::POST, url, Body::empty(), &header).await;
    println!("{}", res);
    Ok(())
}
```

In this snippet, we, as described above, prepare the oauth header with all the parameters we need and pass it to the `make_request` function, which just puts it into the `Authorization` header and calls the provided URL. We also provide a callback_url, which is the url JIRA will call back to if a user authenticates at their site.

```rust
async fn make_request(
    method: Method,
    url: &str,
    req_body: Body,
    oauth_header: &str,
) -> String {
    let client = init_client();
    let req = Request::builder()
        .method(method)
        .uri(url)
        .header("content-type", "application/json")
        .header("Authorization", oauth_header)
        .body(req_body)
        .unwrap();
    let res = client.request(req).await.unwrap();
    let body = to_bytes(res.into_body()).await.unwrap();
    std::str::from_utf8(&body).unwrap().to_owned()
}
```

Here we simply make an HTTP request and return the stringified body. In the case of the request token we'll get an `oauth_token` and an `oauth_token_secret` back, which we just print to the console here.

With this code, you'd usually send the user to JIRA to authenticate. Manually, you can just create a url like this:

```bash
https://some-jira.atlassian.net/plugins/servlet/oauth/authorize?oauth_token=some_token
```

From there, you'll be redirected to the `redirect-url` provided in the initial call, with an `oauth_token` and an `oauth_verifier` as query parameters. These two values can then be used to finally get the access token.

Getting the `access_token` is similar to the other requests. We add the `oauth_token` and `oauth_verifier` to the request, sign it, and we'll get an access token, secret key, refresh token and timestamps for their expiry back.

```rust
async fn get_access_token(
    private_key: &[u8],
    oauth_token: &str,
    oauth_verifier: &str,
) -> Result<()> {
    let nonce = get_nonce();
    let timestamp = get_timestamp();
    let url = "https://some-jira.atlassian.net/plugins/servlet/oauth/access-token";

    let mut params = BTreeMap::new();
    params.insert(OAUTH_CONSUMER_KEY.to_owned(), CONSUMER_KEY.to_owned());
    params.insert(OAUTH_NONCE.to_owned(), nonce.clone());
    params.insert(OAUTH_VERSION.to_owned(), OAUTH_1.to_owned());
    params.insert(OAUTH_TIMESTAMP.to_owned(), timestamp.to_string());
    params.insert(OAUTH_SIGNATURE_METHOD.to_owned(), SIG_METHOD.to_owned());
    params.insert(OAUTH_TOKEN.to_owned(), oauth_token.to_owned());
    params.insert(OAUTH_VERIFIER.to_owned(), oauth_verifier.to_owned());

    let encoded_url = encode(url);
    let parameters = create_params(&params);
    let encoded_parameters = encode(&parameters);
    let combined = format!("POST&{}&{}", encoded_url, encoded_parameters);

    let b64 = sign(&private_key, &combined);

    params.insert(OAUTH_SIGNATURE.to_owned(), b64);

    let header = create_header_params(&params);

    let res = make_request(Method::POST, url, Body::empty(), &header).await;
    println!("{}", res);
    Ok(())
}
```

Alright, now that we have our `access_token`, we can finally make an authenticated request to JIRA. We'll try a request with several additional query parameters, because these have to be handled in a similar way to the oauth parameters.

Let's jump into it. We'll need the `private key` and the `oauth token` for all requests. At first, we need to parse the URL to get a url with and without query parameters. We also need the query parameters themselves.

The next step is to create the parameter map again with the oauth parameters and, if there are any, the additional query parameters. Again, everything is url encoded and then the signature is calculated.

However, for the Authorization header, we don't use the additional query parameters, only the oauth parameters, so the query params are removed from the parameter map again. But from then on, we just add the signature, create the auth header and we can make the request.

This is what all of this looks like in code (in the example of a GET request):

```rust
async fn make_oauth_request(private_key: &[u8], oauth_token: &str) -> Result<()> {
    let url = "https://some-jira.atlassian.net/rest/api/latest/search?jql=project+in+(10000)+order+by+key+asc&startAt=0&maxResults=100&fields=id,key,summary";

    let parsed_url = Url::parse(url).expect("url is valid");
    let query = parsed_url.query_pairs();
    let url_without = url_without_query(parsed_url.clone());

    let nonce = get_nonce();
    let timestamp = get_timestamp();

    let mut params = BTreeMap::new();
    params.insert(OAUTH_CONSUMER_KEY.to_owned(), CONSUMER_KEY.to_owned());
    params.insert(OAUTH_NONCE.to_owned(), nonce.clone());
    params.insert(OAUTH_VERSION.to_owned(), OAUTH_1.to_owned());
    params.insert(OAUTH_TIMESTAMP.to_owned(), timestamp.to_string());
    params.insert(OAUTH_SIGNATURE_METHOD.to_owned(), SIG_METHOD.to_owned());
    params.insert(OAUTH_TOKEN.to_owned(), oauth_token.to_owned());

    for (key, value) in query {
        params.insert(key.into_owned(), value.into_owned());
    }

    let parameters = create_params(&params);

    let encoded_url = encode(&url_without);
    let encoded_parameters = encode(&parameters);
    let combined = format!("GET&{}&{}", encoded_url, encoded_parameters);

    let b64 = sign(&private_key, &combined);

    for (key, _) in query {
        params.remove(key.as_ref());
    }

    params.insert(OAUTH_SIGNATURE.to_owned(), b64);

    let header = create_header_params(&params);

    let res = make_request(Method::GET, url, Body::empty(), &header).await;
    println!("{}", res);

    Ok(())
}
```

That's basically it. In our example, we call the endpoints like this (you might want to comment out all but one request at a time and add the resulting tokens to the next one):

```rust
const FRAGMENT: &AsciiSet = &NON_ALPHANUMERIC.remove(b'.').remove(b'-').remove(b'_');
type Error = Box<dyn std::error::Error + Send + Sync + 'static>;
type Result<T> = std::result::Result<T, Error>;

#[tokio::main]
async fn main() -> Result<()> {
    let private_key = fs::read("./private_key.pem").unwrap();
    get_request_token(&private_key).await?;
    get_access_token(&private_key, "oauth_token", "oauth_verifier").await?;

    make_oauth_request(&private_key, "oauth_token").await?;
    Ok(())
}
```

However, in a real implementation, you'd have user interaction between `get_request_token`, as well as a callback endpoint, which automatically calls the `get_access-token` with the result of the user interaction.

The full code can be found [here](https://github.com/zupzup/oauth1test).

## Conclusion

Although OAuth 1.0a, especially with a less used encryption algorithm is not necessarily something anyone *wants* to use, but there are still cases and APIs where you won't be able to avoid it.

This was a fun exercise and a great learning opportunity, to see how OAuth 1.0 clients work under the hood and while the above is certainly nothing production-ready, it might be a good starting point if you need something comparable.

#### Resources

* [Code Example](https://github.com/zupzup/oauth1test)
* [node-oauth](https://github.com/ciaranj/node-oauth)
