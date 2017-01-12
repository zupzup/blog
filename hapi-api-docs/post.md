If you have ever been in a position where you had to consume a **third party API**, be it from another team in your company or from some open source project which is maintained halfway around the world, it's likely you encountered some kind of **API documentation** along the way.

Now depending on the quality of this documentation, your job was likely either a breeze or absolute hell. Everyone who has been in this situation will know that documentation is important, not only for the success of your current project, but also for the success of every single, small or big, project which will use your code or API in the future.

Yet, although, in my experience, this is known to the majority of professional software developers, it happens regularly that documentation doesn't get done at all, or not enough emphasis is placed on it during development.

Especially in recent times, the reasoning behind this has little to with technical aspects in my experience, but rather the development process and business process attached to developing software. People argue that they want to **iterate quickly**, or that they're just in the process of **building an MVP** for validating a business idea.

Now there isn't anything wrong with these arguments - if we're not even sure if the thing we're building is a valuable thing, why bother investing in possible future consumers of its API? I believe this approach of pushing documentation and testing back a little is actually OK for a in-depth documentation as well as for high-scale end-to-end testing.

In this post, however, I want to talk about one specific kind of documentation inherent in creating complex, distributed systems - **API docs**. I believe that, especially when using a microservice architecture, having documentation for these services' interfaces early on in the development cycle provides great benefits regarding collaboration and iteration speed.

Having API docs can, for example, be quite valuable for getting early feedback from another team, which will have to use the API at some point in the future. The documentation, especially if it's accessible to everyone involved, can also be a useful aid for discussion around complex interactions and workflows in order to find possible problems.

This power of being able to detect misunderstandings and straight up errors in thinking early is why, in my opinion, API docs should be available from the time you write the first endpoint.

In the next few paragraphs, I will provide some examples and insight into the process I like to use when creating well documented **Web APIs** using Node.js.

We will cover:

* [hapi.js](https://hapijs.com/)
  * my go-to Node.js web framework for robust and scalable web-applications
* [joi](https://github.com/hapijs/joi)
  * schema validation library for JavaScript objects 
* [hapi-swaggered](https://github.com/z0mt3c/hapi-swaggered) & [hapi-swaggered-ui](https://github.com/z0mt3c/hapi-swaggered-ui)
  * plugins for automatically creating a swagger-ui documentation from your hapi.js routes 

Let's roll!

## hapi.js & joi

There are countless ways to build web-applications with Node.js, but my favorite way to do this over the last few years was definitely with the great `hapi.js`. It was created by Walmart in order to sustainably handle the immense load of their services while still being able to iterate quickly. With Walmart backing this project and using it for their own high-scale services, you know it will scale (think Black Friday).

The thing I probably like most about `hapi` is its impeccable documentation, which is always up-to-date (with very frequent new releases) and its sane approach in developing it further.

There are several great tutorials on using `hapi` on the web and in their [tutorials](https://hapijs.com/tutorials), but just to give you a taste of what it looks like, consider this example:

```javascript
const Hapi = require('hapi');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply(`Hello, ${request.params.name}!`);
    }
});

server.start((err) => {
    if (err) {
        throw err;
    }
    console.log(`Server running at: ${server.info.uri}`);
});
```

There are of course endless ways to extend this example within the `hapi` ecosystem like logging, authentication, static content, caching, etc. spanning the whole world of the web.

However, one thing I particularly like and also use in a lot of other JavaScript projects is [joi](https://github.com/hapijs/joi), which is a schema validator for JavaScript objects. Now that, by itself, isn't all that exciting, however it can be quite useful with things like form validation and such.

The really cool thing when using `hapi` and `joi` together is that you can plug it directly into `hapi` routes for automatic request and response validation. Consider the `/{name}` route from the example above, let's say we want to make sure that the name is not longer than 10 characters:

```javascript
server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply(`Hello, ${request.params.name}!`);
    },
    config: {
        validate: {
            params: {
                name: joi.string().max(10) 
            }
        }
    }
});
```

 That's it. And we can also do this with path parameters, query parameters, headers and the request payload. If the request doesn't pass validation, `hapi` will respond with a well formed error like this:
 
 ```json
 {
     "error": "Bad Request",
         "message": "the length of name must be less than or equal to 10 characters long",
         "statusCode": 400,
         "validation": {
             "keys": [
                 "name"
             ],
             "source": "params"
         }
 }
 ```

 These errors are automatically generated based on the `joi` schema and can be customized using the whole power of `joi`, which is considerable.

 Another thing we can do is response validation. So we can specify, for each status code, how our response should look like. In the following example, we define a basic error object and extend it for our validation error. Then, we define how the responses for the different possible status codes should look like: 

```javascript
const success = joi.object({
    success: joi.boolean().truthy().required(),
});

const error = joi.object({
    statusCode: joi.number().required(),
    error: joi.string(),
    message: joi.string(),
});

const validationError = error.keys({
    validation: joi.object().optional(),
});


server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply(`Hello, ${request.params.name}!`);
    },
    config: {
        validate: {
            params: {
                name: joi.string().max(10) 
            }
        }
    }
    response: {
        status: {
                200: success,
                400: validationError,
                500: error
            },
    },
});
```

Now this, by itself, is already pretty powerful and can catch a lot of bugs when testing and building our application. However, we can use these **pseudo type annotations** even further to document the API we are building. For this purpose, we can use [hapi-swaggered](https://github.com/z0mt3c/hapi-swaggered) and [hapi-swaggered-ui](https://github.com/z0mt3c/hapi-swaggered-ui).

## hapi-swaggered & hapi-swaggered-ui 

There are several packages available to create a `swagger` documentation from `hapi` routes, but I only used `hapi-swaggered` so far and have been very happy with it. If you don't know `swagger`, I'll just refer you to their [website](http://swagger.io/) - it's basically a framework for doing all kinds of things regarding REST APIs. 

The thing we are interested in is [swagger-ui](http://swagger.io/swagger-ui/), which allows us to visualize our API in an interactive way. A [demo](http://petstore.swagger.io/) is also available.

This is pretty cool and sufficient to provide the benefits I outlined at the start of the post. This UI is easy to navigate and one really gets a feel of the API at hand.

Alright, so integrating `hapi-swaggered` and `hapi-swaggered-ui` is simple enough with `hapi`s well thought out plugin system:

```javascript
const hapi = require('hapi');
const swaggered = require('hapi-swaggered');
const swaggeredUI = require('hapi-swaggered-ui');
const vision = require('vision');
const inert = require('inert');

const server = new hapi.Server();
server.connection({ port: config.get('port'), labels: ['api'] });

server.register([
        inert,
        vision,
        {
            register: swaggered,
            options: {
                info: {
                    title: 'My Cool API',
                    description: 'API documentation for my cool API',
                    version: '1.0',
                },
            },
        },
        {
            register: swaggeredUI,
            options: {
                title: 'My Cool API',
                path: '/docs',
                swaggerOptions: {
                    validatorUrl: null,
                },
            },
        }
], (err) => {
    if (err) {
        throw err;
    }

    server.start((err) => {
        if (err) {
            throw err;
        }
        console.log(`Server running at: ${server.info.uri}`);
    })
});
```

There are several, well documented config options, but this default configuration should provide all we need for now. We basically have a `/docs` route, where we can view a `swagger-ui` visualization of our routes.

Now of course, we need to label the routes we want to be shown in our API docs with the `api` label specified above. We can also add some metadata to the routes:

```javascript
server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply(`Hello, ${request.params.name}!`);
    },
    config: {
        tags: ['api'],
        description: 'Says hello!',
        notes: 'Some important notes when using this',
        validate: {
            params: {
                name: joi.string().max(10) 
            }
        }
    }
});
```

Once labeled, the route will be visible in the `swagger-ui` with all the available schema data parsed. For reference you can check out the swagger petstore [demo](http://petstore.swagger.io/#!/pet/addPet) on how this might look like.

The interface will show, depending on your `joi` schemas, how the data the endpoint expects should look like, as well as the responses.

Another cool thing you can do with `joi` is annotate your schemas with metadata like this:

```javascript
const validationError = error.keys({
    validation: joi.object().optional(),
}).meta({ className: 'ValidationError' });
```

Which gets parsed and included in `swagger-ui`, giving you nice, reusable **type definitions**.

For further configuration regarding `joi`, `hapi` and `hapi-swaggered` I refer to their respective documentation, which should satisfy your every need.

## Conclusion 

This post provided an example of how to automatically generate an interactive API documentation from the existing endpoints of a Node.js web-application. If you're not into Node.js, there are similar libraries and tools for other languages and web frameworks not covered in this post.

I believe that due to the simple setup and configuration of this approach and the immense power of the schema definitions provided by the interaction of these tools, the benefits of doing this far outweigh the extra effort needed to get it to work.

Especially with Node.js, which shines when it comes to iteration speed, but definitely has some trade-offs regarding robustness and maintenance, the approach of well documented validation schemas, which are checked at runtime has, in my opinion, considerable benefits.

Also, after setting this process up for one project, you can simply re-use it in other projects with the same stack, because the configuration will most likely be quite similar.

That's it. Have fun documenting you APIs! :)

#### Resources

* [hapi.js](https://hapijs.com/)
* [swagger](http://swagger.io/)
* [hapi-swaggered](https://github.com/z0mt3c/hapi-swaggered)
* [hapi-swaggered-ui](https://github.com/z0mt3c/hapi-swaggered-ui)
* [joi](https://github.com/hapijs/joi)
