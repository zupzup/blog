Local Port Forwarding as well as Network Tunnelling are useful tools for debugging, navigating infrastructure and much, much more. They're frequently used by sysadmins and ops-engineers, but also provide some nice use-cases for developers.

In this post, we will take a look at how simple it is to do local port forwarding with just the Go standard library and how powerful this concept can be as a building block for more interesting use-cases.

The basic idea in the following example is to send traffic on some local port, which is forwarded to another machine, which then sends it to some service. 

This could be useful, for example, if this service is only accessible from the forwarding machine, but not locally. 

Of course, there are two components to this. A `local` part and a `remote` part. However, these two parts do the same thing, they accept some `TCP` traffic on a port and forward it to a specified target.

As an example, we could try to forward HTTP traffic from our local browser to a remote nginx instance via our tool. We can even simulate this locally, without a remote server.

First, we start an nginx instance on port 8181 (e.g.: with Docker) like this:

```bash
docker run -d -p 8181:80 nginx
```

And then we start two versions of our program:

```bash
go run main.go --target 127.0.0.1:8181 --port 1337
```

This is the `remote` server, which accepts traffic on port `1337` and forwards it to the nginx instance.

```
go run main.go --target 127.0.0.1:1337 --port 1338
```

This is the `local` service, which accepts traffic on `1338`, e.g. from a browser, and forwards it to our `remote` service.

With a setup like this, it should be possible for traffic on port `1338` to go through both applications to nginx on port `8181` and back again seamlessly.

And, of course, the same thing also works over the network. For example, one could connect to a remote server via SSH on the `remote` service and provide this service to the `local` service on some given TCP port.

The same concept works for any TCP-based protocol and we have all the building blocks we need to build ourselves a tunnelling protocol. Pretty Neat!

Alright, now that we have a rough idea of what we're trying to build here, let's look at a simple implementation:

## Example

First off, we use the `flag` package to parse the incoming `target` and `port` parameters.

```go
var (
    target string
    port   int
)

func init() {
    flag.StringVar(&target, "target", "", "target (<host>:<port>)")
    flag.IntVar(&port, "port", 1337, "port")
}
```

Then, we start a server on the given port, so clients can connect to our service (`1337` and `1338` in the above example).

```go
    incoming, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
    if err != nil {
        log.Fatalf("could not start server on %d: %v", port, err)
    }
    fmt.Printf("server running on %d\n", port)
```

We listen for incoming connections and accept the first one, creating our client. This version only works for one connected client, but it wouldn't be hard to change this to a multi-client solution by creating a goroutine for each connected client. 

```go
    client, err := incoming.Accept()
    if err != nil {
        log.Fatal("could not accept client connection", err)
    }
    defer client.Close()
    fmt.Printf("client '%v' connected!\n", client.RemoteAddr())
```

Finally, we connect to the specified target server (`nginx` and the `remote` service in the example).

```go
    target, err := net.Dial("tcp", target)
    if err != nil {
        log.Fatal("could not connect to target", err)
    }
    defer target.Close()
    fmt.Printf("connection to server %v established!\n", target.RemoteAddr())
```

Alright, we have a server which accepts incoming connections and is connected to the target server. Now we need a way to move the traffic between `client` and `target`.

Luckily, both `client` and `target` are of type `net.Conn` and satisfy both the `io.Reader` and `io.Writer` interfaces. So, passing the incoming traffic to the outgoing connection and vice versa is trivial using `io.Copy`.

```go
    go func() { io.Copy(target, client) }()
    go func() { io.Copy(client, target) }()
```

If we would start the program like this however, it would instantly run through and quit after the first connection. For this purpose, we introduce a `stop` channel, which blocks at the end of the program and reacts to an `os.Signal`.

This way the application will continue running until e.g.: `CTRL+C` is pressed.

```go
func main() {
    ...flag code...

    signals := make(chan os.Signal, 1)
    stop := make(chan bool)
    signal.Notify(signals, os.Interrupt)
    go func() {
        for _ = range signals {
            fmt.Println("\nReceived an interrupt, stopping...")
                stop <- true
        }
    }()

    ...forwarding code...

    <-stop
}
```

And that's it.

If we run this application as specified in the beginning of the post, with one `local` and one `remote` part and navigate to `http://localhost:1338` we can see that the traffic is passed through successfully. 

Although this is just a fairly useless HTTP proxy at this state, there are lots of things one could accomplish by extending this simple idea.

For example, you could monitor, multiplex, filter, manipulate or encrypt traffic... endless possibilities! :)

## Conclusion

This was another nice little example made possible by the power of Go's Reader and Writer interfaces.

Also, this example only uses a bare minimum of functionality, all from the standard library, without being overly verbose, which speaks for Go's networking primitives.

Have fun forwarding! :)

#### Resources

* [Go io package](https://golang.org/pkg/io/)
* [Go net package](https://golang.org/pkg/net/)
* [Go os package](https://golang.org/pkg/os/signal/)
* [Full Code](https://gist.github.com/zupzup/14ea270252fbb24332c5e9ba978a8ade)

