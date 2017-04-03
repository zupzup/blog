In Go, NodeJS and many other modern programming languages, the task of creating Web-Applications can be achieved using only the Standard Library and maybe a few microlibraries for utility. This is a nice attribute of these languages, because of the ever increasing size, complexity and also modularity of modern applications.

We live a in a world where hundreds or thousands of containerized microservices are continously deployed, spawned and stopped again within very short intervals. In an environment like that, a small footprint is key. Another benefit to this reduced footprint is the reduced inherent complexity of these systems. Every single additional external dependency means higher mental load as well as possible security or performance issues later on and the inescapable maintenance burden of keeping them up-to-date. 

Traditionally, Java's success story happened in the enterprise sector with business-rritical, robust monolithic applications. In this blog post, We'll find out if the Java Standard Library has kept up with the modern trends and if it's possible to write a lightweight Web Application only using the Java SE JDK 8 Standard Library. 

I believe, that the bloat and high complexity of contemporal Java Web Applications (e.g.: using Spring) is not the only way to do things and that neither Java nor Spring are to blame, but rather the mindset of developers and the "way we've always built things".

Let's see what we can do with just the Java Standard Library!

## Zero Dependencies Web App in Java

**Setup**

We will only create one file called `WebApp.java`, which can be run using `javac WebApp.java && java WebApp`. We can also package this into a naked openJDK Docker container, to really make sure there are no additional libraries on our classpath.

**com.sun.net.httpserver.HttpServer**

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


**Utilities / Constants / DataStore**


**Handlers**




## Conclusion 

As we saw in the above Code-Example, it is entirely possible to write a functional Web Application using only the Java SE Standard Library. Granted, it's not as convenient or fancy as in other languages and it may not even be fit for serious use, but the fact that it was easily possible to do should enable us to think in a more open way about the possibilities to write lightweight applications in Java. 

So Java, the JVM or even Spring (which is actually very modular) are certainly not the reason why Java Web Applications are often so much more complex than they need to be. There is another way. 

Over the last couple of years, several popular and battle-tested micro-frameworks for Java started popping up. One such example is [Spark](http://sparkjava.com/), which provides a modern interface for building web apps with minimal footprint and overhead. If you've been stuck writing bloated, huge Java apps for some time, I'd encourage you to try out one of there microframeworks and take it for a spin - maybe for a small sideproject - and see how it feels.

Java can be lightweight and simple, if you let it. :)

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/java-0-deps-webapp)
* [Slides](https://docs.google.com/presentation/d/1b645UwekJmlUrurGFPdUWQMJ_Dm9GUf2TgJ_3dgrg8E/edit?usp=sharing)
* [Java Http Server](http://www.java2s.com/Code/Java/JDK-6/LightweightHTTPServer.htm)
* [Spark](http://sparkjava.com/)
* [Pippo](http://www.pippo.ro/)
* [Jooby](http://jooby.org/)
* [Jodd](http://jodd.org/0)
* [RestX](http://restx.io/)
