Network Tunneling, what we want to build, what it will look like

`go run main.go -target 127.0.0.1:8181 -p 1337`
`go run main.go -target 127.0.0.1:1337 -p 1338`

## Example 

First, we use the `flag` package to parse our incoming `target` and `port` parameters:

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

Then, we start a server on the given port, so another client can connect to us:

```go
func main() {
	incoming, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		log.Fatalf("err on starting server-socket on %d: %v", port, err)
	}
	fmt.Printf("server running on %d\n", port)
```

Then, we listen for connections and accept them:

```go
	client, err := incoming.Accept()
	defer client.Close()
	if err != nil {
		log.Fatal("err on client connect", err)
	}
	fmt.Printf("client '%v' connected!\n", client.RemoteAddr())
```

Finally, we also connect to the specified target:

```go
	target, err := net.Dial("tcp", target)
	if err != nil {
		log.Fatal("err on target connect", err)
	}
	fmt.Printf("connection to server %v established!\n", target.RemoteAddr())
	defer target.Close()
```

Now we have a server, accepting incoming connections and are connected to our target server as well. Now we need a way to move the traffic between `client` and `target`.

Luckily, both `client` and `target` are of type `net.Conn` and satisfy both the `io.Reader` and `io.Writer` interfaces. So passing the incoming traffic to the outgoing connection and vice versa is trivial using `io.Copy`:

```go
	go func() { io.Copy(target, client) }()
	go func() { io.Copy(client, target) }()
}
```

If we would start this program however, it will instantly run through and quite, because we started goroutines for our copying. For this purpose, we introduce a `stop` channel, which reacts to an `os.Signal`. So the application will continue running until e.g.: CTRL+C is pressed.

```go
func main() {
    ...flag code...

	signals := make(chan os.Signal, 1)
	stop := make(chan bool)
	signal.Notify(signals, os.Interrupt)
	go func() {
		for _ = range signals {
			fmt.Println("\nReceived an interrupt, stopping services...")
			stop <- true
		}
	}()

    ...tunneling code...

    <-stop
}
```

And that's it.

If we would run this like TODO..this and this would happen.

Possibilities for use-cases (monitoring, splitting, multiplexing, filtering...) Extending with encryption etc. etc.

## Conclusion

Another cool example using the power of Go's Reader and Writer interfaces. Also, this example only uses a bare minimum of functionality all from the standard library, without being overly verbose, which speaks for the networking primitives available in the standard library.


#### Resources

* [SomeResources](http://someUrl.com)

