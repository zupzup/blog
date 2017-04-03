In Go, NodeJS and many other modern programming languages, the task of creating Web-Applications can be achieved using only the Standard Library and maybe a few micro libraries for utility. This is a nice attribute of these languages, because of the ever-increasing size, complexity and also modularity of modern applications.

We live a in a world where hundreds or thousands of containerized microservices are continuously deployed, spawned and stopped again within very short intervals. In an environment like that, a small footprint is key. Another benefit to this reduced footprint is the reduced inherent complexity of these systems. Every single additional external dependency means higher mental load as well as possible security or performance issues later and the inescapable maintenance burden of keeping it up-to-date. 

Traditionally, Java's success is prevalent in the enterprise sector with business-critical, robust, but often monolithic applications. In this blog post, we'll find out if the Java Standard Library has kept up with the modern trends and if it's possible to write a lightweight Web Application using only the Java SE JDK 8 Standard Library. 

I believe that the bloat and high complexity of current Java Web Applications (e.g.: using Spring) is not the only way to do things and that neither Java nor Spring are to blame, but rather the mindset of developers and the "way we've always built things in Java".

Let's see what we can do with just the Java Standard Library!

## Zero Dependencies Web App in Java

**Setup**

We will only create one file called `WebApp.java`, which can be run using `javac WebApp.java && java WebApp`. We can also package this into a naked openJDK Docker container, to make sure there are no additional libraries on our classpath.

With that in place, the first step is to start a WebServer. In Go or NodeJS, there is a powerful `http` package in the standard library, which is both widely used and well documented. In Java, the situation is quite a bit different.

In fact, I talked to several seasoned Java developers, who have been creating Java Web Applications for several years and not a single one knew that there was an HTTP-Server in the Java Standard Library.

But there is: [com.sun.net.httpserver](https://docs.oracle.com/javase/8/docs/jre/api/net/httpserver/spec/com/sun/net/httpserver/package-summary.html)

**com.sun.net.httpserver.HttpServer**

This HTTPServer implementation has been around since Java 1.6 and even has a pretty nice interface and some convenience methods.

This is what it looks like in our application:

```java
public class WebApp {
    public static void main(String[] args) throws IOException {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
        server.createContext("/", new LandingPageHandler());
        server.createContext("/post", new PostHandler());
        server.createContext("/json", new JSONHandler());
        server.createContext("/favicon.ico", new IgnoreHandler());

        server.setExecutor(Executors.newCachedThreadPool());
        server.start();

        System.out.println("Server started on port 8080" );
    }
}
```

So, we create a server, register some routes and then start the server. This works very similar to other languages.

For each of these routes, we provide a `Handler` which implements the `HttpHandler` interface. This interface provides a method called `handle`, which receives an `HttpExchange` object.

The `HttpExchange` object encapsulates a complete HTTP interaction with request and response.

Alright, so how does such a `Handler` look like?

**Handlers**

First, we need a handler for serving our static `index.html` template:

```java
class LandingPageHandler implements HttpHandler {
    public void handle(HttpExchange exchange) throws IOException {
        String requestMethod = exchange.getRequestMethod();
        System.out.println(requestMethod + " /");
        if (requestMethod.equalsIgnoreCase("GET")) {
            exchange.getResponseHeaders().set(Constants.CONTENTTYPE, Constants.TEXTHTML);
            exchange.sendResponseHeaders(200, 0);
            OutputStream responseBody = exchange.getResponseBody();
            responseBody.write(Constants.getIndexHTML(null));
            responseBody.close();
        } else {
            new NotImplementedHandler().handle(exchange);
        }
    }
}
```

This is pretty straightforward I think. Granted, it's a bit verbose, but if we created a `HttpGetHandler` interface as well as a `writeHTML` convenience method, this could be 2 lines of code, so it's really not a big deal. 

Of course we also want to interact with the Web Application, e.g. using `POST`:

```java
class PostHandler implements HttpHandler {
    public void handle(HttpExchange exchange) throws IOException {
        String requestMethod = exchange.getRequestMethod();
        System.out.println(requestMethod + " /post");
        if (requestMethod.equalsIgnoreCase("POST")) {
            String body = new BufferedReader(
                    new InputStreamReader(
                            exchange.getRequestBody()
                    )
            ).lines().collect(Collectors.joining("\n"));
            String[] parts = body.split("=");
            String name = null;
            if (parts.length > 1) {
                name = parts[1];
            }
            exchange.getResponseHeaders().set(Constants.CONTENTTYPE, Constants.TEXTHTML);
            exchange.sendResponseHeaders(200, 0);
            OutputStream responseBody = exchange.getResponseBody();
            responseBody.write(Constants.getIndexHTML(name));
            responseBody.close();
        } else {
            new NotImplementedHandler().handle(exchange);
        }
    }
}
```

This is even more verbose, but that's mostly due to my dummy payload parsing logic, which could be easily abstracted in a `parseRequestBody` utility method, which returns a map of parameters with their assigned values.

Other than that, our `HttpExchange` object provides everything we need and we pass the given `name` to our `template` in `getIndexHTML`.

**JSON**

In a modern web application we would most likely create some REST endpoints serving JSON, which would then be used by a mobile application or by some flavor-of-the-month-.js JavaScript Single Page Application.

Sadly, native JSON support is not in the Java Standard Library and has been pushed back even further into JDK 10 last time I checked. So marshalling and unmarshalling JSON is quite tedious without a nice library such as GSON or Jackson, because we have to use the `Nashorn` JavaScript engine within the JVM with all the boundary-crossing trouble that brings with it.

So JSON is definitely an area where using a low footprint library makes a lot of sense instead of relying on the *native* method.

**NotImplemented / IgnoreHandler**

In cases where we have no response or simply want to tell a client that we don't implement the endpoint which was called, we can easily pass the exchange along to another `HttpHandler`.

Of course it's a bit bothersome to do this for each and every case, which is usually handled by some nice web-framework, but at least it's simple to do and can be abstracted without much fuss.

```java
class NotImplementedHandler implements HttpHandler {
    public void handle(HttpExchange exchange) throws IOException {
        exchange.sendResponseHeaders(501, -1);
        exchange.close();
    }
}

class IgnoreHandler implements HttpHandler {
    public void handle(HttpExchange exchange) throws IOException {
        exchange.sendResponseHeaders(204, -1);
        exchange.close();
    }
}
```

That's it. This bit of code is just a crude example and definitely not fit for use in any serious application, but I believe it's interesting and it showcases that Java has all the tools one needs to write low-overhead web apps.

The whole code for this example can be found [here](https://github.com/zupzup/java-0-deps-webapp/blob/master/WebApp.java), with a dummy DataStore and even some simple native JSON parsing.

## Conclusion 

As we saw in the above Code-Example, it is entirely possible to write a functional Web Application using only the Java SE Standard Library. Granted, it's not as convenient or fancy as in other languages and it may not even be fit for serious use, but the fact that it was easily possible to do should enable us to think in a more open way about the possibilities of writing lightweight applications in Java. 

So Java, the JVM or even Spring (which is very modular) are certainly not the reason why Java Web Applications are often so much more complex than they need to be. There is another way. 

Over the last couple of years, several popular and battle-tested micro-frameworks for Java started popping up. One such example is [Spark](http://sparkjava.com/), which provides a modern interface for building web apps with minimal footprint and overhead. If you've been stuck writing bloated, huge Java apps for some time, I'd encourage you to try out Spark or one of the microframeworks mentioned below. Take it for a spin - maybe for a small side project - and see how it feels.

Java can be lightweight and simple, if you let it. :)

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/java-0-deps-webapp)
* [Slides from my Talk on this subject at Java User Group Graz](https://docs.google.com/presentation/d/1b645UwekJmlUrurGFPdUWQMJ_Dm9GUf2TgJ_3dgrg8E/edit?usp=sharing)
* [Java Http Server](http://www.java2s.com/Code/Java/JDK-6/LightweightHTTPServer.htm)
* Java Microframeworks
  * [Spark](http://sparkjava.com/)
  * [Pippo](http://www.pippo.ro/)
  * [Jooby](http://jooby.org/)
  * [Jodd](http://jodd.org/0)
  * [RestX](http://restx.io/)
