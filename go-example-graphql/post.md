TBD
graphql cool concept, especially with graph-like data
versioning / versionless api through control over schema and queries
[graphql](http://graphql.org/)
[graphql-go](https://github.com/graphql-go/graphql)
[graphql-go-handler](https://github.com/graphql-go/handler)
[graphiql](https://github.com/graphql/graphiql)
[jsonplaceholder](http://jsonplaceholder.typicode.com/)

example to wrap an existing API with a thin graphql layer as a first step to move to a graphql api
wrap jsonplaceholder and convert some of it to be able to be queried by graphql

## Example 

The goal is to fetch `posts` and `comments` from `jsonplaceholder` and end up with a way to fetch `posts` by ID and, if the API consumer wishes to also fetch the comments, to nest the comments into the `post`.

We define a `fetchPostByiD(id)` function, which calls `http://jsonplaceholder.typicode.com/posts/${id}` and transforms the resulting JSON to a `Post`. Of course, there is also a `fetchCommentsByPostID(post.ID)` helper function, whcih does the same for comments, by fetching the data from `http://jsonplaceholder.typicode.com/posts/${id}/comments` and transforming it to `[]Comment`.

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

Then we go on to create our graphQL schema. We start by defining the `queryType`, which is the root of our schema.

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

The root query type has only one field - the `post`. This field is defined by `postType`, which we will look at next and takes only one argument: `id`. Posts are resolved by taking the `id` from `p.Args` and passing it to `fetchPostsByID`, returning the fetched and transformed `posts` as well as any error.

Next, we define the `postType`, which is a bit more interesting. We add a `comments` field with a custom resolver:

We map the `post` fields from data model to graphQL and  add the `comments` field. The `Resolve` function is only called, if the client explicitly wants to fetch `comments`. 

To resolve comments, we access the "parent" of this query by using `p.Source`, which yields us an instance of `*Post` - the fetched `post`. Using the `id` of this `post`, we can fetch the comments for it.

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

We instantiate a `graphql` schema and pass it to `graphql-go-handler`, which is an http-middleware handling graphql-queries. Then, we simply start an `http` server with the returned handler routed to `/graphql`.

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
        log.Fatalf("failed to create new schema, error: %v", err)
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

After starting the server, we can use `GraphiQL` to query for a post with a certain `id`, specifying the fields we are interested int:

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

If we omit the `comments` from the query, the request to fetch the comments is never made and we simply get the selected post as a response.

The whole code for this example can be found [here](https://github.com/zupzup/example-go-graphql).

## Conclusion 

TBD
i'm sure there are nicer, more concise ways to create schema, but this is optimized for clarity
graphql great
`go-graphql` looks good and transferring knowledge from JS to Go worked well
future post, hope to deal with graphql subscriptions

#### Resources

* [Full Code on Github](https://github.com/zupzup/example-go-graphql)
* [graphql](http://graphql.org/)
* [graphql-go](https://github.com/graphql-go/graphql)
* [graphql-go-handler](https://github.com/graphql-go/handler)
* [graphiql](https://github.com/graphql/graphiql)
* [jsonplaceholder](http://jsonplaceholder.typicode.com/)
