Much has been written and said about the work of art that are the `io.Reader` and `io.Writer` interfaces. Simple, yet powerful - just as Go itself.

In this post I want to showcase another part of the Go standard library that I find to be both simple and powerful - [io.Pipe](https://golang.org/pkg/io/#Pipe).

```go
pr, pw := io.Pipe()
```

According to the docs, `io.Pipe` creates a synchronous in-memory pipe, which can be used to connect code expecting `io.Reader` with code expecting `io.Writer`.

Upon invocation, `io.Pipe()` returns a `PipeReader` and a `PipeWriter`. They are connected (hence the pipe), so that everything written to the `PipeWriter` can be read from the `PipeReader`.

The following three examples show use-cases of `io.Pipe`, its versatility and the way of thinking and composing I/O it enables us to do. 

Let's get started!

## Example 1: JSON to HTTP Request

This is the go-to example one usually sees when it comes to `io.Pipe`. We encode some data as `JSON` and want to send it to a web endpoint via `http.Post`. Unfortunately (or rather fortunately), the JSON encoder takes an `io.Writer` and the http request methods expect an `io.Reader` as input, so we can't just plug them together.

Of course we could always create intermediate `[]byte` representations, but that is neither memory efficient nor particularly elegant. This is where `io.Pipe` comes in:

```go
pr, pw := io.Pipe()

go func() {
    // close the writer, so the reader knows there's no more data
    defer pw.Close()

    // write json data to the PipeReader through the PipeWriter
    if err := json.NewEncoder(pw).Encode(&PayLoad{Content: "Hello Pipe!"}); err != nil {
        log.Fatal(err)
    }
}()

// JSON from the PipeWriter lands in the PipeReader
// ...and we send it off...
if _, err := http.Post("http://example.com", "application/json", pr); err != nil {
    log.Fatal(err)
}
```

First, we encode some struct `PayLoad` to JSON and write the data to the `PipeWriter` created by invoking `io.Pipe`.
Afterwards, we create a http POST request, which gets its data from the `PipeReader`. That `PipeReader` gets filled with the data written to the `PipeWriter`.

Important to note here is that we have to encode asynchronously to prevent a deadlock, because we would write without a reader if we didn't.

This practical example showcases the versatility of `io.Pipe` very well. It really incentivizes gophers to build components using `io.Reader` and `io.Writer`, without having to worry about them being used together.

## Example 2: Split up Data with TeeReader 

I found another very cool way of using `io.Pipe` together with `TeeReader` (read: T-Reader) in [@rodaine](https://twitter.com/rodaine)'s great [blog post](http://rodaine.com/2015/04/async-split-io-reader-in-golang/) about asynchronously splitting an `io.Reader`.

In *Solution #4*, he describes the use-case of using a video-file and simultaneously transcode it to another format and uploading that, while also uploading the original file. All with minimal overhead and completely in parallel.

Based on this solution, I tried to capture the gist of it with the following example:

```go
pr, pw := io.Pipe()

// we need to wait for everything to be done
wg := sync.WaitGroup{}
wg.Add(2)

// we get some file as input
f, err := os.Open("./fruit.txt")
if err != nil {
    log.Fatal(err)
}

// TeeReader gets the data from the file and also writes it to the PipeWriter
tr := io.TeeReader(f, pw) 

go func() {
    defer wg.Done()
    defer pw.Close()

    // get data from the TeeReader, which feeds the PipeReader through the PipeWriter
    _, err := http.Post("https://example.com", "text/html", tr)
    if err != nil {
        log.Fatal(err)
    }
}()

go func() {
    defer wg.Done()
    // read from the PipeReader to stdout
    if _, err := io.Copy(os.Stdout, pr); err != nil {
        log.Fatal(err)
    }
}()

wg.Wait()
```

My example is of course simplified in that it doesn't use channels for propagating errors and results, but the underlying concept is quite similar - we have some kind of input `io.Reader`, a file in this case and create a `TeeReader`, which returns a *Reader* that writes to the *Writer* you provide it everything it reads from the *Reader* you provide it.

Now we start two goroutines, one which just prints the data to stdout and another one which sends it to an HTTP endpoint. The `TeeReader` uses the `io.Pipe` to split up the given input. When the `TeeReader` is consumed, those same bytes are also received by the `PipeReader`.

Pretty cool, ha?

## Example 3: Piping the output of Shell commands

I stumbled over this [gist](https://gist.github.com/ifels/10392762) recently, which combines `io.Pipe` with `os.Exec` in a nice way. Basically, it does what most task runners in CI services like Jenkins or Travis CI do, which is execute some shell command and show its output on some website.

I tried to encapsulate the general pattern behind it in this short snippet here:

```go
pr, pw := io.Pipe()
defer pw.Close()

// tell the command to write to our pipe
cmd := exec.Command("cat", "fruit.txt")
cmd.Stdout = pw

go func() {
    defer pr.Close()
    // copy the data written to the PipeReader via the cmd to stdout
    if _, err := io.Copy(os.Stdout, pr); err != nil {
        log.Fatal(err)
    }
}()

// run the command, which writes all output to the PipeWriter
// which then ends up in the PipeReader
if err := cmd.Run(); err != nil {
    log.Fatal(err)
}
```

First, we define our command - in this case, we just `cat` a file called `fruit.txt`, which will just spit out the contents of the file on `stdout`. Then, and this is important, we set the command's `stdout` to our `PipeWriter`.

So we redirect the output of the `Command` to our pipe, which, as before, will make it possible to read it through our `PipeReader` at another point. In this rather contrived case, that point is just a goroutine where we dump the results of cat to stdout (which it would have done anyways), but I think it's easy to imagine doing something nifty here like exporting the results of the command somewhere or flushing it to a webpage as seen in this [gist](https://gist.github.com/ifels/10392762), where we'd need an `io.Writer` as input.

## Conclusion

I hope these examples helped to convince you of the many opportunities opened by using `io.Pipe` together with nice abstractions which expect either `io.Reader` or `io.Writer`. Not only does `io.Pipe` enable seamless composition of components based on best practices, it's also quite flexible with the use of `TeeReader`, which points the vast possibilities of using `io.Pipe` in custom-made I/O handling pipelines in both a readable and scalable way.

Of course this post only scratched the surface on this topic, as it didn't handle the inherent gotchas with this approach nor error handling, but I plan to remedy this by a post or two on these and some more advanced topics in the future.

Have fun pipin'! :)

#### Resources

* [Code on GitHub](https://github.com/zupzup/golang-io-pipe)
* [io.Pipe](https://golang.org/pkg/io/#Pipe)
* [TeeReader](https://golang.org/pkg/io/#TeeReader)
* [Split with TeeReader](http://rodaine.com/2015/04/async-split-io-reader-in-golang/)
* [Cmd to HTTP Response gist](https://gist.github.com/ifels/10392762)

