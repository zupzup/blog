This blog is created using a static site generator I wrote a few weeks ago, which I will hopefully be able to publish on Github soon and which I will cover in a future post.

I was pretty satisfied with the outcome. The next step was hosting and I decided to use [AWS](https://aws.amazon.com/) - **S3** for storage and **Cloudfront** for getting the content to my readers (that's you!). Everything went pretty well and due to the ease of use and documentation of these services I had everything set up in no time.

Then came the moment of truth - my first post. I was excited and swiftly uploaded everything using the web interface, only to realize that my files didn't have any cache headers set, which is of course unacceptable for a statically generated website tuned for performance even with a very slow connection.

Another problem was that I had to manually invalidate several Cloudfront objects in order for users to see my new content. 

After setting the cache headers for all my static assets manually everything was fine, but I felt a bit uneasy going forward as I would have to do this for all the files I wanted to upload in the future and then invalidate all relevant URLs of my blog.
So I searched for a tool which would help me with this, but didn't find anything I liked right away...so...I decided to write a tool myself using Go! :D

Now...I'm sure there are tons of ready to use solutions for this out there hidden in some SDK or Software Package, but this problem seemed small enough to fit into a nice, lean, single-purpose tool as well as provide a few challenges to improve my knowledge of Go while hacking on it.

So off I went!

## Idea & Concept

I started with the following features in mind:

* lightweight CLI
* simple authentication 
* upload to S3
  * configurable cache headers for different file sets
  * configurable source folder with an option to ignore some files
* cloudfront invalidation
  * configurable URLs to invalidate
* dry-run option
  * to test a config before starting to upload / invalidate (which can create costs)

Go is great at creating lightweight CLIs, even without any additional libraries or frameworks. The `flag` package and convenient I/O is usually sufficient for small projects like this.

For configuration a simple `config.yml` file should suffice with regular expressions for filtering files and file sets for the headers.

Authentication would be handled either by providing it via the configuration file or via environment variables, which is simple enough and puts the responsibility to handle the tool securely on the user.

So far so good!

## Implementation

Looking at the feature set, I decided on the following structure:

* **main.go**
* package **invalidate**
  * **invalidate.go**
* package **upload**
  * **upload.go**

This way everyone can use the isolated `invalidate` and `upload` packages on their own if needed.

### main.go 

There's really not much happening here. I use the `flag` package to parse commandline flags and read the `config.yml` file into the following data structure:

```go
type Config struct {
    Auth struct {
        Accesskey string
        Key       string
    }
    S3         upload.Config
    Cloudfront invalidate.Config
}
```

With `S3` and `Cloudfront` representing the configuration objects for my two `worker` packages.

After parsing the configuration file, `main.go` simply calls the other two packages with the given commandline options and respective configuration. It also passes in a `logger`, so the packages don't have to write to Stdout. Another nice benefit of this is that a `silent` flag can be implemented by passing in the `ioutil.Discard` writer, which throws away all messages written to it.

If an error happens, it is just logged and the program exits.

The full source of `main.go` can be found [here](https://github.com/zupzup/static-aws-deploy/blob/master/main.go)

### upload/upload.go

This is where it gets a little more interesting. The `upload` package has two public functions:

* `ParseFiles`
  * Creates and returns a map from filepath to headers based on the configuration
* `Do`
  * Uploads the files as specified in the map created with `ParseFiles`

The configuration object for the `upload` package looks like this:

```go
type Header map[string]string

type Files map[string][]Header

type Config struct {
    Bucket struct {
        Name      string
        Accesskey string
        Key       string
    }
    Parallel int
    Source   string
    Ignore   string
    Metadata []struct {
        Regex   string
        Headers []Header
    }
}
```

`ParseFiles` walks the `Source` folder, ignoring all files matched by the `Ignore` regex and adds the `Headers` given in `Metadata` if the file matches the `Regex`. What comes out is a `Files` object, which is a map from the filepath to the configured `Headers` for the file.

The second function, `Do` takes the `Files` object and concurrently (specified by the `Parallel` option) calls `uploadFile` for each of the entries, setting the appropriate headers for each file.

In `uploadFile`, the `Multipart` file is created and sent to the AWS S3 `PUT` endpoint. Initially, I wanted to do this using `io.Pipe`, to efficiently stream the files with the minimal possible memory footprint, but the AWS API for chunked upload is not compatible with the default implementation of the Go standard library.

Although it would certainly be interesting to solve this, it seems very much like premature optimization (especially considering a static blog with filesizes in the low kilobytes). 

Request authentication is done using the great [github.com/smartystreets/go-aws-auth](https://github.com/smartystreets/go-aws-auth) package, which makes it as simple as this:

```go
req, err := http.NewRequest(...)
awsauth.Sign(req, awsauth.Credentials{
    AccessKeyID:     config.Bucket.Accesskey,
    SecretAccessKey: config.Bucket.Key,
})
```

Of course there are a ton of errors which can happen during this I/O heavy process, which are all handled accordingly. In case of a `dry-run`, the files which would have been uploaded are printed on the screen.

The full source of `upload.go` can be found [here](https://github.com/zupzup/static-aws-deploy/blob/master/upload/upload.go)

### invalidate/invalidate.go

The `invalidate` package has only one public function:

* `Do`
  * creates the XML payload and sends it to the AWS Cloudfront API

The configuration object for `invalidate` looks like this:

```go
type Config struct {
    Distribution struct {
        Id        string
        Accesskey string
        Key       string
    }
    Invalidation []string
}
```

The invalidation is a bit simpler than the upload portion. First, the XML payload is created using the amazing [github.com/beevik/etree](https://github.com/beevik/etree) package and is then sent to the `invalidation` endpoint of the Cloudfront API.

Request authentication works the same way as in the `upload` package. In case of a `dry-run`, URLs which would be invalidated are printed to the screen.

The full source of `invalidate.go` can be found [here](https://github.com/zupzup/static-aws-deploy/blob/master/invalidate/invalidate.go)

**That's it - Done! You can see the result [here](https://github.com/zupzup/static-aws-deploy)**

## Conclusion

In my opinion, it's always a fun learning experience to solve small automation problems like this yourself. I also prefer lean, single-purpose tools over bloated software which does just about everything if you just configure it right (which can take as long as writing a new tool yourself sometimes...).

Solving this simple problem was also a nice way to improve my Go skills with a few interesting challenges such as concurrently uploading files, building an easy to use CLI and working with a weird XML-based API.

The tool described in this post does exactly what I need right now: It enables me to update my blog with one simple command.

If the need arises, I may add some more features to it in the future, but for now I'm pretty happy with it. If there are other people who find it useful - even better! :)

BTW: The fact that you can read this, means that this blogpost was successfully uploaded using the tool described in this post! :)

#### Resources

* [static-aws-deploy on Github](https://github.com/zupzup/static-aws-deploy)
* [S3 REST API](http://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
* [Cloudfront API](http://docs.aws.amazon.com/AmazonCloudFront/latest/APIReference/Welcome.html)
* [AWS Go SDK](https://github.com/aws/aws-sdk-go/)

