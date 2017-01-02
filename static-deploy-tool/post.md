This blog is created using a static site generator I wrote a few weeks ago, which I will hopefully be able to publish on Github soon and which I will cover in a future post.

I was pretty satisfied with the outcome. The next step was hosting and I decided to use [AWS](https://aws.amazon.com/) `S3` for storage and `Cloudfront` for getting it to the users (that's you!). Everything went pretty well and due to the ease of use and documentation of these services I had everything set up in no time.

Then came the moment of truth - my first post. I was excited and swiftly uploaded everything using the web interface, only to realize that my files didn't have any cache headers set, which is of course unacceptable for a statically generated website tuned for performance even with a very slow connection.
Another problem was that I had to manually invalidate the `Cloudfront` objects, in order for users to see my new content. 

After setting the cache headers for all my static assets manually everything was fine, but I felt a bit uneasy going forward as I would have to do this for all the files I wanted to upload in the future and then invalidate all relevant URLs of my blog.
So I searched for a tool which would help me with this, but didn't find anything I liked right away...so...I decided to write a tool myself using Go!

Now...I'm sure there are tons of ready to use solutions for this out there hidden in some SDKs and software packages, but this problem seemed small enough to fit into a nice, lean, single-purpose tool as well as provide a few challenges to improve my knowledge of Go while hacking on it.

So off I went!

## Idea

I started with the following features in mind:

* lightweight CLI
* simple and secure authentication 
* upload to S3
  * configure cache headers for different filesets
  * configure source with option to ignore some files
* cloudfront invalidation
  * configure URLs to invalidate
* dry-run option
  * to test a config before starting to upload / invalidate (which can create costs)

Go is great at creating lightweight CLIs, even without any additional libraries or frameworks. The `flag` package and convenient I/O is usually sufficient for small projects like this.
Authentication would be handled either by providing it via the configuration file or via environment variables, which is the way it is done in most tools I use.


## Implementation


## Conclusion


#### Resources

* [static-aws-deploy on Github](https://github.com/zupzup/static-aws-deploy)
* [S3 REST API](http://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
* [Cloudfront API](http://docs.aws.amazon.com/AmazonCloudFront/latest/APIReference/Welcome.html)
* [AWS Go SDK](https://github.com/aws/aws-sdk-go/)

