It seems like [GraphQL](http://graphql.org/) is being used more and more and having been present at some of the first public talks about it within the react ecosystem a couple of years back, I'm not surprised.

The concept works out from several perspectives, with graph-like data, multiple distributed teams and highly versioned APIs as well as regarding type safety and documentation. GraphQL looks like a good fit for many different applications.

This post's goal is not to introduce you the basics of GraphQL, but rather to see it in action in a realistic scenario. When planning to move an existing REST API to GraphQL, it makes sense to first introduce a translation-layer, to make a smooth transition.

In this post, we will use [jsonplaceholder](http://jsonplaceholder.typicode.com/) as the API we will wrap with GraphQL. There are a couple of libraries for graphQL in Go and for this example, [graphql-go](https://github.com/graphql-go/graphql) and [graphql-go-handler](https://github.com/graphql-go/handler) will be used.

Our goal is to fetch `posts` and `comments` from `jsonplaceholder` and end up with a way to fetch `posts` by ID and, if the API consumer wishes to also fetch the comments, to nest the comments into the `post` via GraphQL.

Let's get started.

## Implementation

First, we define data models for `Post` and `Comment`:

```go
type Post struct {
    UserID int    `json:"userId"`
    ID     int    `json:"id"`
    Title  string `json:"title"`
    Body   string `json:"body"`
}

type Comment struct {
    PostID int    `json:"postId"`
    ID     int    `json:"id"`
    Name   string `json:"name"`
    Email  string `json:"email"`
    Body   string `json:"body"`
}
```

We also define a `fetchPostByiD(id)` function, which calls `http://jsonplaceholder.typicode.com/posts/${id}` and transforms the resulting JSON to a `Post`. Of course, there is also a `fetchCommentsByPostID(post.ID)` helper function, which does the same for comments, by fetching the data from `http://jsonplaceholder.typicode.com/posts/${id}/comments` and transforming it to `[]Comment`.

Then we go on to create our graphQL schema. We start by defining the `queryType`, which is the root of our schema:

```go
func createQueryType(postType *graphql.Object) graphql.ObjectConfig {
    return graphql.ObjectConfig{Name: "QueryType", Fields: graphql.Fields{
        "post": &graphql.Field{
            Type: postType,
            Args: graphql.FieldConfigArgument{
                "id": &graphql.ArgumentConfig{
                    Type: graphql.NewNonNull(graphql.Int),
                },
            },
            Resolve: func(p graphql.ResolveParams) (interface{}, error) {
                id := p.Args["id"]
                v, _ := id.(int)
                log.Printf("fetching post with id: %d", v)
                return fetchPostByiD(v)
            },
        },
    }}
}
```

The root query type has only one field - the `post`. This field is defined by `postType`, which we will look at shortly. It takes only one argument called `id`.

Posts are resolved by taking the `id` from `p.Args` and passing it to `fetchPostsByID`, returning the fetched and transformed `Post` as well as any error.

Next, we define the `postType`, which is quite interesting. We basically just map the `post` fields from the data model to graphQL types, but we also add the `comments` field. The comment's `Resolve` function is only called, if the client explicitly wants to fetch them. 

To resolve comments, we access the "parent" of this query by using `p.Source`, which yields us an instance of `*Post` - the fetched `post`. Using the `id` of this `post`, we can fetch the comments for it:

```go
func createPostType(commentType *graphql.Object) *graphql.Object {
    return graphql.NewObject(graphql.ObjectConfig{
        Name: "Post",
        Fields: graphql.Fields{
            "userId": &graphql.Field{
                Type: graphql.NewNonNull(graphql.Int),
            },
            "id": &graphql.Field{
                Type: graphql.NewNonNull(graphql.Int),
            },
            "title": &graphql.Field{
                Type: graphql.String,
            },
            "body": &graphql.Field{
                Type: graphql.String,
            },
            "comments": &graphql.Field{
                Type: graphql.NewList(commentType),
                Resolve: func(p graphql.ResolveParams) (interface{}, error) {
                    post, _ := p.Source.(*Post)
                    log.Printf("fetching comments of post with id: %d", post.ID)
                    return fetchCommentsByPostID(post.ID)
                },
            },
        },
    })
}
```

The only type left to define in the schema is the `commentType`, which is pretty boring, as it only maps fields of the data model to graphQL types:

```go
func createCommentType() *graphql.Object {
    return graphql.NewObject(graphql.ObjectConfig{
        Name: "Comment",
        Fields: graphql.Fields{
            "postid": &graphql.Field{
                Type: graphql.NewNonNull(graphql.Int),
            },
            "id": &graphql.Field{
                Type: graphql.NewNonNull(graphql.Int),
            },
            "name": &graphql.Field{
                Type: graphql.String,
            },
            "email": &graphql.Field{
                Type: graphql.String,
            },
            "body": &graphql.Field{
                Type: graphql.String,
            },
        },
    })
}
```

Alright, our schema is defined and the only thing left is to put it all together.

We instantiate a `graphQL` schema and pass it to `graphql-go-handler`, which is an http-middleware which helps us to deal with graphQL queries. Then we simply start an `http` server with the returned handler routed to by `/graphql`.

This is what it looks like:

```go
func main() {
    schema, err := graphql.NewSchema(graphql.SchemaConfig{
        Query: graphql.NewObject(
            createQueryType(
                createPostType(
                    createCommentType(),
                ),
            ),
        ),
    })
    if err != nil {
        log.Fatalf("failed to create schema, error: %v", err)
    }
    handler := gqlhandler.New(&gqlhandler.Config{
        Schema: &schema,
    })
    http.Handle("/graphql", handler)
    log.Println("Server started at http://localhost:3000/graphql")
    log.Fatal(http.ListenAndServe(":3000", nil))
}
```

Alright, that's it!

After starting the server, we can use [GraphiQL](https://github.com/graphql/graphiql) to query for a post with a certain `id`, specifying the fields we are interested in:

```
query {
  post(id: 5) {
    userId
    id
    body
    title
    comments {
      id
      email
      name
    }
  }
}
```

Resulting in the following response:

```json
{
  "data": {
    "post": {
      "userId": 1,
      "id": 5,
      "title": "...",
      "body": "...",
      "comments": [
        {
          "id": 21,
          "email": "...",
          "name": "..."
        },
        ...
      ]
    }
  }
}
```

If we omit `comments` from the query, the request to fetch the comments is never made and we simply get the selected `post` as a response.

The complete example code can be found [here](https://github.com/zupzup/example-go-graphql).

## Conclusion 

This example showed how to transform an existing REST API to GraphQL using a thin Go layer. The library I used for this, `graphql-go` worked nicely, provided solid documentation and good examples to follow. Also, it closely mirrors the `graphql-js` API, which I was already familiar with, which made the conversion to Go a lot easier.

There are surely more concise and fancier ways to define a schema such as this, but due to the introductory nature and my unfamiliarity with graphQL in Go I went for this solution, which is focused, above anything else, on clarity.

GraphQL seems like it is here to stay and for good reason. I hope to be able to touch on GraphQL subscriptions in a future blog post, as well as some other, more advanced use-cases. :)

#### Resources

* [Full Code on Github](https://github.com/zupzup/example-go-graphql)
* [graphql](http://graphql.org/)
* [graphql-go](https://github.com/graphql-go/graphql)
* [graphql-go-handler](https://github.com/graphql-go/handler)
* [graphiql](https://github.com/graphql/graphiql)
* [jsonplaceholder](http://jsonplaceholder.typicode.com/)
