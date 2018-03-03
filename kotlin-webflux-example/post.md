TBD

## Example 

```java
@SpringBootApplication
class KotlinWebfluxDemoApplication

fun main(args: Array<String>) {
    val app = SpringApplication(KotlinWebfluxDemoApplication::class.java)
    app.webApplicationType = WebApplicationType.REACTIVE
    app.run(*args)
}
```

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

```java
data class Comment(
        val postId: Int,
        val id: Int,
        val name: String,
        val email: String,
        val body: String
)

data class LightComment(
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
```

```java
@Service
class APIService {
    companion object : KLogging()

    fun fetchComments(postId: Int): Flux<Comment> = fetch("posts/$postId/comments").bodyToFlux(Comment::class.java)

    fun fetchPosts(): Flux<Post> = fetch("/posts").bodyToFlux(Post::class.java)

    fun fetch(path: String): WebClient.ResponseSpec {
        val client = WebClient.create("http://jsonplaceholder.typicode.com/")
        return client.get().uri(path).retrieve()
    }
}
```

```java
@RestController
@RequestMapping(path = ["/api"], produces = [ APPLICATION_JSON_UTF8_VALUE ])
class APIController(
        private val apiService: APIService
) {

    companion object : KLogging()

    @RequestMapping(method = [RequestMethod.GET])
    fun getData(): Mono<ResponseEntity<List<Response>>> {
        return apiService.fetchPosts()
                .filter { it -> it.userId % 2 == 0 }
                .take(20)
                .map { post -> apiService.fetchComments(post.id)
                        .parallel(4)
                        .runOn(Schedulers.parallel())
                        .map { comment -> LightComment(email = comment.email, body = comment.body) }
                        .sequential()
                        .collectList()
                        .zipWith(post.toMono()) }

                .flatMap { it -> it }
                .map { result -> Response(
                        postId = result.t2.id,
                        userId = result.t2.userId,
                        title = result.t2.title,
                        comments = result.t1
                ) }
                .collectList()
                .map { body -> ResponseEntity.ok().body(body) }
                .toMono()
    }
}
```

The whole code for this example can be found [here](https://github.com/zupzup/kotlin-webflux-example).

## Conclusion 

TBD

#### Resources

* [Full Code on Github](https://github.com/zupzup/kotlin-webflux-example)
* [kotlin](http://kotlinlang.org/)
* [spring boot](https://projects.spring.io/spring-boot/)
* [webflux](https://docs.spring.io/spring-framework/docs/5.0.5.BUILD-SNAPSHOT/spring-framework-reference/web-reactive.html#spring-webflux)
* [reactor](https://projectreactor.io/)
* [jsonplaceholder](http://jsonplaceholder.typicode.com/)
