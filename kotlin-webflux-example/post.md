I have recently been doing quite a bit of [kotlin](http://kotlinlang.org/) at [timeular](https://timeular.com/) and must say I like it a lot so far.

Also, with the new [Spring Boot 2.0.0](https://projects.spring.io/spring-boot/), the kotlin support got improved and there is now a reactive option to the `Spring MVC` web framework called [Webflux](https://docs.spring.io/spring-framework/docs/5.0.5.BUILD-SNAPSHOT/spring-framework-reference/web-reactive.html#spring-webflux).

In this post, we will take a look at how to create a small, Kotlin-based Spring-Boot application using Webflux. We will create a Web Application with a single `GET /api` route, which fetches `posts` and `comments` from [jsonplaceholder](http://jsonplaceholder.typicode.com/) in parallel and transforms them to a combined output.

Webflux uses the [Reactor](https://projectreactor.io/) library underneath, which has two important types: `Flux` and `Mono`. A `Flux` is a reactive `Publisher` for 0 to n values, whereas a `Mono` is limited to 0 or 1 value. A more detailed explanation can be found [here](https://github.com/reactor/reactor-core).

There is also a new functional DSL for specifying routes in Webflux, but in this example, we will stick to the good old `@RestController` to define our route.

Let's get started.

## Example 

First, let's create a new Spring Boot application at [https://start.spring.io/](https://start.spring.io/) selecting the `Reactive Web` dependency. With this application template, we can start our implementation by configuring the Spring Boot application to be a `reactive` web application:

```java
@SpringBootApplication
class KotlinWebfluxDemoApplication

fun main(args: Array<String>) {
    val app = SpringApplication(KotlinWebfluxDemoApplication::class.java)
    app.webApplicationType = WebApplicationType.REACTIVE
    app.run(*args)
}
```

And we also create a `WebConfig` class, which adds `cors` mappings, has the `@EnableWebFlux` annotation and sets up our dependency injection config.

```java
@Configuration
@EnableWebFlux
@ComponentScan("org.zupzup.kotlinwebfluxdemo")
class WebConfig: WebFluxConfigurer {
    override fun addCorsMappings(registry: CorsRegistry) {
        registry.addMapping("api/**")
    }
}
```

With the boilerplate out of the way, let's create some data models. Kotlin provides the great `data class` for this purpose, which let's us define value classes in a highly succinct way.

In this example, we will need a `Post` and `Comment` for the data source and a `Response` class for our transformed response:

```java
data class Comment(
        val postId: Int,
        val id: Int,
        val name: String,
        val email: String,
        val body: String
)

data class Post(
        val userId: Int,
        val id: Int,
        val title: String,
        val body: String
)

data class Response(
        val postId: Int,
        val userId: Int,
        val title: String,
        val comments: List<LightComment>
)

data class LightComment(
        val email: String,
        val body: String
)
```

In order to fetch data from `jsonplaceholder`, we create an `APIService`, which uses the reactive `WebClient` integrated into Webflux.

This is quite nice, as it enables us to easily transform our responses to a `Flux` or `Mono` of the model represented by the returned JSON.

```java
@Service
class APIService {
    fun fetchComments(postId: Int): Flux<Comment> = fetch("posts/$postId/comments").bodyToFlux(Comment::class.java)

    fun fetchPosts(): Flux<Post> = fetch("/posts").bodyToFlux(Post::class.java)

    fun fetch(path: String): WebClient.ResponseSpec {
        val client = WebClient.create("http://jsonplaceholder.typicode.com/")
        return client.get().uri(path).retrieve()
    }
}
```

That's all we need for fetching the data and transforming it to reactive versions of our models. Pretty cool. The `WebClient` works well and provides all kinds of utility functionality you would expect.

The last and most important/interesting part is our handler. In this small example we will implement the whole logic right in the controller for simplicity.

The goal is to fetch 20 `posts` with an even `id` from `jsonplaceholder` and then, for each `post` in parallel, fetch its `comments` and transforming the `posts` and `comments` into a nested data-structure like this:

```json
[
    {
        "postId": 1,
        "userId": 2,
        "title": "...",
        "comments":
        [
            {
                "email": "...",
                "body": "..."
            },
            ...
        ]
    },
    ...
]
```

Let's see how that works:

```java
@RestController
@RequestMapping(path = ["/api"], produces = [ APPLICATION_JSON_UTF8_VALUE ])
class APIController(
        private val apiService: APIService
) {
    @RequestMapping(method = [RequestMethod.GET])
    fun getData(): Mono<ResponseEntity<List<Response>>> {
        return apiService.fetchPosts()
                .filter { it -> it.userId % 2 == 0 }
                .take(20)
                .parallel(4)
                .runOn(Schedulers.parallel())
                .map { post -> apiService.fetchComments(post.id)
                        .map { comment -> LightComment(email = comment.email, body = comment.body) }
                        .collectList()
                        .zipWith(post.toMono()) }

                .flatMap { it -> it }
                .map { result -> Response(
                        postId = result.t2.id,
                        userId = result.t2.userId,
                        title = result.t2.title,
                        comments = result.t1
                ) }
                .sequential()
                .collectList()
                .map { body -> ResponseEntity.ok().body(body) }
                .toMono()
    }
}
```

Ok, quite a few things are happening here and if you're not used to functional programming or reactive sequences, this might not seem very intuitive at first - don't worry about it, it's the same for everyone.

We start off by fetching the `posts` using our injected `apiService`. Then we `filter` this reactive stream of `post` elements, using only `posts` with even ids and taking only the first 20 elements (`take(20)`). So far, so good.

To fetch the comments with explicit parallelism of `4`, we add `.parallel(4) and .runOn(Schedulers.parallel())` to the stream. Later on, we have to call `.sequential()` again to wait for all the values fetched in parallel, otherwise we couldn't stuff them together in a List, which we need to be able to `zip` the `comments` with the `post`.

Just to be clear, the `parallel(4)` is not necessary to fetch the comments concurrently, but rather explicitly controls the parallelism of the concurrent and asynchronous processing. As [@hpgrahsl](https://twitter.com/hpgrahsl) pointed out to me on twitter, the idiomatic, recommended way for this use-case would be to just use `flatMap`.

In any case, the next step is to `map` the `posts` to another reactive sequence. In this new sequence, we call `fetchComments` with the given `post.id` to fetch a post's comments. The goal is to map these `comments` to `LightComment`, throwing away some of the data we don't want to display and to `zip` them up with the `post` data, so we can create the above mentioned data structure later on.

Now we have a sequence of `Tuple2<List<LightComment, Post>>` from the `zipWith` operation. We use `flatMap` to convert the stream from a sequence of `Monos` to a sequence of just our zipped data, which we then simply `map` to the `Response` model.

All that's left now is `collecting` all the values to a list and mapping that list to a `Mono` with a `ResponseEntity` inside it, including our data as the body, which is what the controller expects as a return value.

That's it!

The whole code for this example can be found [here](https://github.com/zupzup/kotlin-webflux-example).

## Conclusion 

Reactive programming, when you're used to the style, can be a lot of fun. There are some great benefits when writing applications in a non-blocking way, especially when dealing with microservice architectures.

There are some downsides to this asynchronous, non-blocking, stream-transforming way of coding like that it's inherently hard to test and debug and that there is a bit of a learning-curve at first, but for the right use-cases the benefits outweigh them in my opinion.

It's been a while since I used Spring Boot or Spring in general and I'm impressed by the progress which happened in the meantime. Webflux, especially with Kotlin, looks like something I will strongly consider for a future service. :)

*Thanks to [@hpgrahsl](https://twitter.com/hpgrahsl) for the invaluable feedback regarding the use of `parallel` and `flatMap`.*

#### Resources

* [Full Code on Github](https://github.com/zupzup/kotlin-webflux-example)
* [kotlin](http://kotlinlang.org/)
* [spring boot](https://projects.spring.io/spring-boot/)
* [webflux](https://docs.spring.io/spring-framework/docs/5.0.5.BUILD-SNAPSHOT/spring-framework-reference/web-reactive.html#spring-webflux)
* [reactor](https://projectreactor.io/)
* [jsonplaceholder](http://jsonplaceholder.typicode.com/)
