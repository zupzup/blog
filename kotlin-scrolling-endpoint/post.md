When fetching a huge amount of data entries (e.g. users, products, posts...), the sheer amount of data is often a limiting factor and can have a negative effect on backend performance (load) as well as on the client experience (huge payload, long loading times).

A good way to solve this problem is to scroll through the data set in smaller chunks. This is also nice when the goal is to load some data up-front and to load the rest later on-demand.

In this post, we will look at an implementation of exactly this mechanism using Kotlin and Spring Boot 2. The data source will just be a list of 1000 randomly generated users and we'll traverse them in chunks of 100. In practice, you would probably only implement this, if the amount of data is much larger, but for this example it's a nice size for testing.

There are multiple ways to implement something like this. One of the core questions is whether the scrolling state is handled by the server, or the client. In this example, we will handle that state on the server (remembering the amount of entries the client already received). This makes the API very clean at the cost of some flexibility.

First, let's get into how we want to use this scrolling API.

## Usage Flow

At the beginning, we need to make an initial Request:

```bash
curl localhost:8080/rest/v1/user/scroll
```

This will trigger the following response (with a random UUID of course):

```json
{"data":{"scrollParam":"94422391-b626-4926-aca9-cc7ae4e6f3a5","users":[...]}}
```

We get some data already from the first request and, more importantly, the `scrollParam`, which is a unique ID identifying our scrolling-state on the backend.

Now, we copy the `scrollParam` in order to use it for subsequent requests, scrolling through the whole user list:

```bash
curl localhost:8080/rest/v1/user/scroll?scrollParam=94422391-b626-4926-aca9-cc7ae4e6f3a5
```

When we finished scrolling through the whole list, we get an empty list back:

```json
{"data":{"scrollParam":"94422391-b626-4926-aca9-cc7ae4e6f3a5","users":[]}}
```

Once we hit an empty list, we remove the scrolling State and any further requests will result in an error:

```json
{"error":{"message":"You must provide an active scrollParam, ...  does not exist"}}
```

The client can either take the empty list or the error as the signal to stop requesting new data.

That's the basic idea behind what we want to build - so let's get coding!

## Code Example

The Spring Boot boilerplate will be omitted here for brevity, but the full code is up on [GitHub](https://github.com/zupzup/kotlin-scrollingdemo)

Ok, first up, the data model for the users:

```java
data class User (
    val id: Int,
    val firstName: String,
    val lastName: String,
    val email: String
)
```

With that out of the way, let's look at the definition of our controller. It waits for requests at `GET /rest/v1/user/scroll`:

```java
@RestController
@RequestMapping("rest/v1/user")
class ScrollingController(
        private val userService: UserService
) {
    companion object : KLogging()

    @GetMapping("/scroll")
    fun listUsers(
            @RequestParam(name = "scrollParam", required = false) scrollParam: String?
    ): ResponseEntity<ResultData<UserScrollingDTO>> {
        return try {
            val result = userService.scrollUsers(scrollParam)
            createSuccess(UserScrollingDTO(
                            scrollParam = result.scrollParam,
                            users = result.users
                        ))
        } catch (e: UserScrollParamNotFoundInCacheException) {
            return createError("You must provide an active scrollParam,
                the scrollParam `$scrollParam` does not exist")
        }
    }
}
```

There is an optional `scrollParam` parameter. Remember when discussing the usage flow above, that the first request won't have a scrollParam, but every subsequent one needs it. So if a client requests the endpoint without a scrollParam, we create a new scrolling state on the backend.

The endpoint calls the `scrollUsers` method in the `userService`, which is the core part of the mechanism. For the service layer, we need two data classes as well:

```java
data class UserScrollingResult(
    var scrollParam: String,
    var users: List<User>
)

data class UserScrollRequest(
    var scrollParam: String,
    var cursor: Int
)
```

The first one is simply a wrapper for the result, to keep a clean separation between the controller and the service layer. The `UserScrollRequest` represents the scrolling state. That's all there is to it really, we save a UUID and the last ID the client received.

For more advanced use-cases, this request object could also include some request parameters for filtering the list. With this, the client would only need to provide these filters at the initial request, but for the rest of the requests, the `scrollParam` would suffice.

It's important to use the `ID` of the last data point the client received and not just to count up, because intermittent changes to the data set could lead to the order of entries being wrong as well as to duplicates.

Before we get into the service methods, some setup needs to happen. In order to efficiently keep the scrolling state (if necessary across several instances of our application), we will put it into a shared cache.

```java
@Service
class UserService {
    private val listOfUsers: List<User> = initializeUserList()
    private val cacheManager: CacheManager = ConcurrentMapCacheManager(CACHE_KEY_SCROLLING)

    private fun initializeUserList(): List<User> {
        val result = ArrayList<User>()
        for (i in 0..1000) {
            result.add(User(
                    i,
                    getRandomString(),
                    getRandomString(),
                    "${getRandomString()}@${getRandomString()}.${getRandomString()}"
            ))
        }
        return result
    }

    private fun getRandomString(): String = RandomStringUtils.randomAlphanumeric(3, 15)
    ...
}
```

In this example, it's a simple in-memory map, but this could easily just be an external redis instance.

After the cache initialization, we also initialize our randomly generated list of users. Nothing fancy happening here, just some random strings thrown together in a `User` object with an ID.

With all the setup out of the way, let's get to the interesting stuff. The `scrollUsers` service method encapsulates the whole mechanism for saving, updating and evicting scrolling state as well as for creating the `UserScrollingResult`:

```java
fun scrollUsers(scrollParam: String?): UserScrollingResult {
    val scrollRequest: UserScrollRequest
    val activeScrollParam = scrollParam ?: UUID.randomUUID().toString()
    scrollRequest = if (scrollParam == null) {
        putUserScrollRequestInCache(activeScrollParam)
    } else {
        getUserScrollRequestFromCache(scrollParam)
    }
    val users = fetchUsers(scrollRequest.cursor)
    if (users.isEmpty()) {
        evictUserScrollRequestFromCache(activeScrollParam)
    } else {
        updateUserScrollRequestCursorInCache(activeScrollParam, users.last().id)
    }
    return UserScrollingResult(activeScrollParam, users)
}
```

Alright, plenty of things happen in the above snippet. First, if we get passed a `scrollParam`, we use it, otherwise we create a new `UUID`.

If the `scrollParam` was not provided, we create a new scrolling State and put it in the cache, otherwise we fetch the existing scrolling state for the given `scrollParam` from the cache:

```java
private fun putUserScrollRequestInCache(
        scrollParam: String
): UserScrollRequest {
    val scrollRequest = UserScrollRequest(
            scrollParam = scrollParam,
            cursor = 0
    )
    cacheManager.getCache(CACHE_KEY_SCROLLING)!!.put(scrollParam, scrollRequest)
    return scrollRequest
}

private fun getUserScrollRequestFromCache(scrollParam: String): UserScrollRequest {
    val value = cacheManager.getCache(CACHE_KEY_SCROLLING)!!.get(scrollParam)
    if (value != null && value.get() is UserScrollRequest) {
        return value.get() as UserScrollRequest
    }
    throw UserScrollParamNotFoundInCacheException(scrollParam)
}
```

If there is no scrolling state in the cache for the given `UUID`, we throw an exception.

Then, with our scrolling state, we fetch the users with the current `cursor`. Now, if the returned list of users is empty, we're at the end of the list and we evict the scrolling state from the cache.

If it is not, we simply update the scrolling state with the new cursor:

```java
private fun updateUserScrollRequestCursorInCache(scrollParam: String, cursor: Int): UserScrollRequest? {
    val value = cacheManager.getCache(CACHE_KEY_SCROLLING)!!.get(scrollParam)
    if (value == null || value.get() !is UserScrollRequest) {
        return null
    }
    val scrollRequest = value.get() as UserScrollRequest
    scrollRequest.cursor = cursor
    cacheManager.getCache(CACHE_KEY_SCROLLING)!!.put(scrollParam, scrollRequest)
    return scrollRequest
}

private fun evictUserScrollRequestFromCache(scrollParam: String) =
        cacheManager.getCache(CACHE_KEY_SCROLLING)!!.evict(scrollParam)
```

After all that, the only thing left is to return a `UserScrollResult` with the `scrollParam` and the list of users to return.

Actually fetching the users, in this contrived case, is just a bit of cursor-counting in our randomly generated user list:

```java
private fun fetchUsers(cursor: Int): List<User> {
    val result = ArrayList<User>()
    if (cursor > listOfUsers.size) return result
    for (user in listOfUsers) {
        if (user.id <= cursor) continue
        if (user.id >= cursor + SCROLL_SIZE) return result
        result.add(user)
    }
    return result
}
```

In a real-world implementation of this, the `fetchUsers` function would probably fetch data from a database or from another webservice.

Alright, that's it!

The whole code for this example can be found [here](https://github.com/zupzup/kotlin-scrollingdemo).

## Conclusion

I really like Kotlin, especially having done Java for some time before. Implementing this mechanism in a relatively close-to-real way didn't take me a lot of time due to the power of Spring Boot and the convenience and conciseness of Kotlin.

After having used Kotlin a lot for the last 6 months for production microservices, I'm still very happy with the choice. :)

#### Resources

* [Full Code Example](https://github.com/zupzup/kotlin-scrollingdemo)
* [kotlin](http://kotlinlang.org/)
* [spring boot](https://projects.spring.io/spring-boot/)
