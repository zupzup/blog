This is the story of a bug hunt within a go microservice, which took quite some time and in the end came down to a rather small detail (as usual). After experiencing issues such as this, one always feels a bit silly because the problem seems so obvious after finding it, but this feeling is usually misguided, as anyone who has ever written production-grade software can probably attest that these things just happen.

At first, I'll describe the service and the problem, then some unsuccessful attempts to find the issue and finally the simple solution.

Let's get started. :)

## The Service

The service in question, we'll call it `event-service` here, is basically an event-routing service, which lets clients subscribe to arbitrary events, which are sent downstream to the clients.

If an event happens (e.g. `somethingCreated`), this event is put on a `rabbitmq` instance and from there fanned out to all instances of this event service. If an instance has a client who subscribed to the incoming event, that event is sent to that client via WebSockets (the client has to initiate that connection before, of course).

The WebSocket communication is generally unidirectional, so events are only pushed down to the clients, never up to the event service from a client.

## The Problem 

After rolling the service out, we started noticing that instances running the service were building up memory slowly, but consistently. Even up to a point that such an instance went out of memory (despite the service being memory-limited) and crashed - fortunately our infrastructure is resilient enough that end-users were not impacted by this.

However, this behaviour was very strange and worrisome - none of the processes running on the instance seemed to take up the memory which was vanishing, but still it kept going up, up, up until the instance would eventually crash. We tried several different configurations to rule out a certain combination of services running on the instance to be responsible for our issues, but without luck.

We also noticed during experimentation that killing the open WebSocket connections (e.g. by restarting the load-balancer or the service itself), made the memory be reclaimed as well, so we assumed it might have something to do with cleaning up the connections, but again, we couldn't find any problems there.

The most confusing thing was, that the memory seemed to go nowhere - no unix command we tried showed us the missing memory and even AWS support couldn't help us find the memory (but they had a lot less context to be fair).

Frustratingly, we were also not able to reproduce this issue on our testing system, even with load-tests blasting the testing system to the ground, we simply couldn't replicate the same behaviour.

Anyway, after trying lots of different tools, commands and asking people for help, we actually went into full trial-and-error mode and that's where I coincidentally stumbled over the problem - the missing WebSocket Read Handler.

## The Simple Solution 

Basically, our setup regarding the WebSocket was pretty standard stuff from the [Gorilla WebSockets](https://github.com/gorilla/websocket) documentation. On authenticating and registering a user, we open a WebSocket connection.

However, as semantically, this connection is only used to push events down to clients and never back up from the clients, we only defined a `writeHandler` on the connection, but no `readHandler` - a fatal mistake.

What we did not factor in, was that clients CAN send messages up the line and also WILL, because they send a regular `ping` message, to ensure the connection is still working. In our load-tests, we didn't think to integrate this `ping` message, which is why we couldn't reproduce the issue. This also explains, why the memory didn't show up in the `event-service` process directly, or on any other docker process.

As it turns out, the continuous pings from the clients filled up the connection buffers, which was indeed visible on `netstat` (unfortunately we didn't pay that stat any attention before), but not on any memory statistics.

So in the end, adding a `read-handler`, which simply discards the messages solved the issue.

```go
func reader(c *Client) {
    for {
        _, _, err := c.conn.ReadMessage()
            if err != nil {
                log.WithField("error", err).Debug("Read error happened - stop reading from connection")
                    close(c.readFailure)
                    break
            }
    }
}
```

In this example code, `c.readFailure` is a channel waiting to be notified of a read failure, which is subscribed to by a goroutine, which then shuts down the whole connection and all running goroutines/channels connected to it, so we don't leak resources.

This subtle change fixed the whole conundrum and while afterwards it seems rather obvious, if you come from a higher-level language such as C#, Java, JavaScript etc., you would probably (as we did) expect there to be a standard read handler, which avoids exactly this situation from happening.

However, this is somewhat documented by gorilla WebSockets and with Go being a bit lower level, it's clearly our mistake for not reading the docs properly and for testing this case in depth.

## Conclusion

Bugs like these seem silly once you found them, but especially with the current complexity of web stacks, these things are bound to happen.

However, having made this experience, this kind of bug will forever be on my radar and I believe sharing such stories makes it less likely that other people waste their time trying to find similar issues.

In any case, I hope this was interesting/helpful for you and that it may save you some time and headaches in the future! :)

#### Resources

* [Gorilla WebSockets](https://github.com/gorilla/websocket)
