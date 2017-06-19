Docker is a very convenient tool for helping with reproducible builds across platforms and for deployment. This does not only hold true for backend-services, but can also be applied to Single Page Applications.

This can make sense, if the static SPA is deployed alongside other dockerized components (Microservices, Database, Cache...), as these systems will often rely on a tool like kubernetes or some other container orchestration tool for deployment and management. 

In such a setup, we might want to package our SPA in a container as well so we can use the same tools as for the other components. This can be achieved by using some HTTP Server as a base-image and just copying in the static files. Using nginx, this could look like this:

```bash
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
COPY . /usr/share/nginx/html
EXPOSE 80
RUN chown nginx.nginx /usr/share/nginx/html/ -R
```

We use the nginx base-image, copy in a custom `nginx.conf` and put our SPA into `/usr/share/nginx/html`. After building and running with this config, we will have a Docker container serving our Single Page Application.

## The Problem

So far so good, but modern JavaScript applications, especially when using react, have a complex build-pipeline to create these static files.
Whether this added complexity is good or bad is not in the scope of this post, but the reality is that frontend developers nowadays use tools like `webpack`, `Babel` and many others to create their applications.

This presents a bit of a problem, as with the approach outlined above, we would still have to build our SPA outside the container, with all the dependencies that build-pipeline needs, crushing our dreams of reproducible builds.

A solution to this dilemma would be to use a base-image which consists of nginx, nodeJS and all other dependencies we need and then build the application inside.

This approach, in addition to being an ugly hack, also has the disadvantage that the resulting image will be quite big, as it contains all the build dependencies and artefacts, which are unnecessary for running the application.

## The Solution

Fortunately, the engineers at Docker came up with a solution to this problem with the release of `17.05`. They call it [Multi-Stage-Builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/) and the idea is, that one can use several `FROM` clauses in the same Dockerfile and use the output of previous `FROM` sections.

So, we could create a build-step, which uses `FROM node:boron` and runs our build-pipeline and then reference the output of this build-step in our second `FROM nginx` part. This way, the application can be built using Docker with all the nice benefits across platforms, but the resulting image only contains whatever is in the last `FROM` section.

A `Dockerfile` for this, with a react application created using [create-react-app](https://github.com/facebookincubator/create-react-app) could look like this:

```bash
FROM node:boron as builder
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY package.json /usr/src/app
RUN npm install
COPY . /usr/src/app
RUN npm run build

FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /usr/src/app/build /usr/share/nginx/html
EXPOSE 80
RUN chown nginx.nginx /usr/share/nginx/html/ -R
```

If we use `docker build` with this config, the application will be built using create-react-app's `npm run build` script and then copied into the nginx container using the `COPY --from=builder` command.

The resulting container will only include nginx, our static files and the config - pretty cool!

By the way, the only reason for using react and create-react-app for this example were to simulate an overly complex build-process with lots of dependencies, this approach will however work with any technology or framework which has a build-step and creates a static website as its output (e.g. a static-site-generator like Hugo, or any other JS framework).

A full code example can be found [here](https://github.com/zupzup/multi-stage-docker-react)

## Conclusion 

I like Docker for development. The Multi-Stage Builds can also be used for Go (and other languages), where one can build the application in the first step and then put the static binary in a very lightweight container afterwards, resulting in a 5-6 MB Docker-image depending on the application and language.

This approach won't make much sense, if one doesn't use Docker already, but in a setup where other services are using it the method described in this post may be helpful for automation. 

I personally am (after 2+ years of react development) not a big fan of the complexity added by the ~30 dev-dependencies which are create-react-app's default and like to use a more lightweight setup for my projects, but I'm afraid the transpiling and overall build-pipeline of JavaScript frameworks is here to stay and Docker can be quite helpful in this regard.

#### Resources

* [example on GitHub](https://github.com/zupzup/multi-stage-docker-react)
* [Docker Multi-Stage Builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/)
* [create-react-app](https://github.com/facebookincubator/create-react-app)
