*This post was originally posted on the [LogRocket](https://blog.logrocket.com/writing-webpack-plugins-rust-using-swc/) blog on 24.05.2023 and was cross-posted here by the author.*

In a [previous post](https://blog.logrocket.com/migrating-swc-webpack-babel-overview/), we covered how to migrate an existing webpack project from Babel to [Speedy Web Compiler (SWC)](https://swc.rs). SWC is a high-performance JavaScript/TypeScript compiler written in Rust, with the goal of speeding up existing build pipelines for modern frontend web projects.

In this article, we’ll look at how to write custom plugins for SWC, which can then be used in webpack builds. By migrating your existing custom webpack and Babel plugins to Rust, you’ll be able to make your webpack builds even faster.

Before we jump in, there is a small disclaimer. At the time of writing, the SWC plugin system is still experimental. If you plan to use your custom SWC plugins in an actual project, you should be prepared to deal with some API instability and potential breaking changes. The APIs described here may change in the upcoming months and years, but the concepts and general approach shared in this article should remain valid.

In this tutorial, we’ll create a simple custom Rust-based SWC plugin and compile it to Wasm, so it can be used in a webpack build using `swc-loader`. We’ll build our simple plugin and import it in a JavaScript project using webpack, configure it as a plugin running with the `[swc-loader](https://github.com/swc-project/swc-loader)` within webpack, and then check that the plugin was run and that it worked.

Let’s get started!

## Setting up the project

For the purposes of this tutorial, we’ll build a very simple plugin that will transform `console.log` statements in the codebase to `console.debug` statements. This plugin may not be particularly useful in real life, but it’s a solid example to demonstrate basic concepts, rather than trying to show how to use SWC to build complex Abstract Syntax Tree (AST)-transformation plugins.

If you’re interested in learning how to build more complex plugins, I recommend checking out the [SWC docs](https://github.com/swc-project/plugins).

Let’s start by installing the SWC command line interface:

```bash
    cargo install swc_cli
```

There is currently a [bug in a dependency](https://swc.rs/docs/plugin/selecting-swc-core#swc_core), so let’s use Rust `nightly-2022-09-23` to build our plugin. We’ll use the below commands to install this version of Rust and set it as the default:

```bash
    rustup install nightly-2022-09-23
    rustup default nightly-2022-09-23-x86_64-unknown-linux-gnu
```

Now, let’s add a Wasm compile target:

```bash
    rustup target add wasm32-wasi
```

Next, we’ll create a new SWC plugin project, like so:

```bash
    swc plugin new --target-type wasm32-wasi rust-swc-webpack-plugin
```

This will create a `rust-swc-webpack-plugin` folder and, inside the folder, it will initialize everything we need to start building our plugin.

## Setting up the SWC plugin

For this tutorial, we’ll use `swc_core` v.0.69. At the time of writing, the Wasm-based SWC plugins are not backward compatible, so it’s important that the `swc_core` version used in our Rust-based plugin is compatible with the `swc/core` version that will be used in the JavaScript project later on:

```bash
    swc_core = { version = "0.69.*", features = ["common", "ecma_utils", "ecma_plugin_transform"] }
```

The SWC CLI creates a `src/lib.rs` file with some available template code. It also sets up all the dependencies and other items relevant to the build process.

Check out the `Cargo.toml` file, as well as the `src/lib.rs` file and `package.json` created by the CLI before moving on, to get a better idea of the starting point for building an SWC plugin.

Let’s start implementing our custom SWC plugin!

## Implementing the SWC plugin

As a first step, let’s take a look in the `lib.rs` file:

```rust
    use swc_core::ecma::{
        ast::Program,
        transforms::testing::test,
        visit::{as_folder, FoldWith, VisitMut},
    };
    use swc_core::plugin::{plugin_transform, proxies::TransformPluginProgramMetadata};
    
    pub struct TransformVisitor;
    
    impl VisitMut for TransformVisitor {
    
    }
```

We have the basic parts of `swc_core` imported, but we need to implement the [VisitMut](https://rustdoc.swc.rs/swc_ecma_visit/trait.VisitMut.html) trait for a given struct called `TransformVisitor`.  With the `VisitMut` trait, we can implement many different methods, which will be called when the AST is traversed. In our example we’ll implement `visit_mut_call_expr`, which will be called whenever a call expression, such as `console.log()`, is encountered.

The above code also includes a `plugin_transform` function and macro; these are some of the basic “machinery” used to wire the plugin up correctly.

We also have a default test harness, so we can use `cargo test` to test our plugin:

```rust
    test!(
        Default::default(),
        |_| as_folder(TransformVisitor),
        boo,
        // Input codes
        r#"console.log("transform");"#,
        // Output codes after transformed with plugin
        r#"console.log("transform");"#
    );
```

From this base, we can start implementing our custom SWC plugin. First, let’s introduce some constants:

```rust
    const CONSOLE: &str = "console";
    const DEBUG: &str = "debug";
    const LOG: &str = "log";
```

Next, we’ll implement our actual plugin logic:

```rust
    impl VisitMut for TransformVisitor {
        noop_visit_mut_type!();
    
        fn visit_mut_call_expr(&mut self, call_expr: &mut CallExpr) {
            call_expr.visit_mut_children_with(self);
    
            if let Callee::Expr(callee) = &mut call_expr.callee {
                if let Expr::Member(member) = &**callee {
                    if let (Expr::Ident(obj), MemberProp::Ident(prop)) = (&*member.obj, &member.prop) {
                        if &obj.sym == CONSOLE && &prop.sym == LOG {
                            *callee = Box::new(Expr::Member(MemberExpr {
                                span: DUMMY_SP,
                                obj: member.obj.to_owned(),
                                prop: MemberProp::Ident(quote_ident!(DEBUG)),
                            }));
                        }
                    }
                }
            }
        }
    }
```

In the above code, `noop_visit_mut_type!();` simply indicates that TypeScript types don’t need to be considered, which reduces the binary size of the plugin.

Then we implement the `visit_mut_call_expr` method of the `VisitMut` trait, which will be called for every call expression encountered in the AST.

We use `visit_mut_children_with` to ensure that our logic is also called for all children nodes of the current node.

Then we implement the actual logic of our plugin. We start by drilling down the AST of the call expression, by getting the `callee` of the expression, and by splitting the expression up into the thing we call something on (`obj`) and the thing we call (`prop`).

With these two elements (`obj` and `prop`), we can check that we only run our logic if we call `log` on `console`.

If this is the case, we update the `callee` to use the `debug` call, instead of the `log` call.

To stay with the scope of this post, I won’t go into too much detail on the types, such as `MemberExpr`, `Expr::Ident`, etc., representing the AST involved here. For more information, check out the [official docs](https://rustdoc.swc.rs/swc_ecma_ast/struct.CallExpr.html).

That’s quite a bit of code for changing a simple property, but keep in mind that we’re literally manipulating the AST while traversing it. In addition, we’re dealing with the entire complexity of ECMAScript and TypeScript.

## Creating the plugin test cases

Hopefully we’ve implemented our logic correctly! Let’s check with some testing.

We’ll use the [test-harness](https://docs.rs/test-harness/latest/test_harness/) macro to create some test cases for our plugin:

```rust
    test!(
        Default::default(),
        |_| as_folder(TransformVisitor),
        log_to_debug,
        // Input codes
        r#"console.log("hello, world");"#,
        // Output codes after transformed with plugin
        r#"console.debug("hello, world");"#
    );
    
    test!(
        Default::default(),
        |_| as_folder(TransformVisitor),
        debug_stays_debug,
        // Input codes
        r#"console.debug("hello, world");"#,
        // Output codes after transformed with plugin
        r#"console.debug("hello, world");"#
    );
    
    test!(
        Default::default(),
        |_| as_folder(TransformVisitor),
        not_interested_in_args,
        // Input codes
        r#"console.debug("log");"#,
        // Output codes after transformed with plugin
        r#"console.debug("log");"#
    );
```

We can use `cargo test` to test our SWC plugin. If everything was implemented correctly, we should see the following output:

```bash
    running 3 tests
    test log_to_debug ... ok
    test not_interested_in_args ... ok
    test debug_stays_debug ... ok
    
    test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Great — things seem to be working!


## Building the SWC plugin

Now that we’ve confirmed that everything is working properly, let’s build our plugin using the following command:

```bash
    cargo build-wasi --release
```

This will create our Wasm file at `target/wasm32-wasi/release/rust_swc_webpack_plugin.wasm`. This is the point where we could actually [publish our plugin](https://swc.rs/docs/plugin/publishing) to npm, so that it could be used conveniently in projects.

With swc-loader, we can use this Wasm file in our webpack configuration in order to use our new plugin in a webpack build.

## Testing the plugin with webpack

Now it’s time to test our SWC plugin with webpack.


### Setup

Let’s start by setting up a basic JavaScript project with webpack by running `npm init` followed by `npm i --save-dev @swc/core swc-loader webpack webpack-cli` in a new folder.

In this example, let’s use the below versions of the various packages. As mentioned previously, the `@swc/core` version is especially important since it has to match up with the `swc_core` version in our Rust project:

```json
    {
    ...
      "devDependencies": {
        "@swc/core": "^1.3.39",
        "swc-loader": "^0.2.3",is already
        "webpack": "^5.77.0",
        "webpack-cli": "^5.0.1"
      }
      ...
    }
```

Next, we’ll create an `src` folder and add a file, called `index.js`, inside it with the following content:

```bash
    console.log("hello, world");
    console.debug("this is already debuged. ");
    console.debug("log");
    console.trace("hello");
```

Here, we simply add a couple of `console.log` statements, so that we can test if our plugin runs and does its job properly.

Next, we’ll create a file called `webpack.config.js` in the root of the project. This file will hold our webpack config.

In the webpack config, we’ll start with the basics:

```json
    module.exports = {
        name: 'swc-plugin-test',
        mode: "production",
        entry: path.join(__dirname, "src", "index.js"),
        output: {
            path: path.join(__dirname, "build"),
            filename: "bundle.js"
        },
    }
```

Here we set a name for the config, set the `mode` to `production`, and set the `entry` point and `output` for the webpack build — basic stuff. Now let’s look at the interesting stuff — how to configure `swc-loader` and a custom plugin:

```json
    ...
        module: {
            rules: [
                {
                    test: /\.js$/,
                    use: [
                        {
                            loader: 'swc-loader',
                            options: {
                                jsc: {
                                    parser: { syntax: 'ecmascript' },
                                    experimental: {
                                        plugins: [["/file/path/to/rust-swc-webpack-plugin/target/wasm32-wasi/release/rust_swc_webpack_plugin.wasm", {}]]
                                    },
                                },
                            },
                        }
                    ],
                    exclude: /node_modules/,
                },
            ],
        },
    }
```

Here we define a `module` for all `.js` files. We also specify that `swc-loader` should be used as a `loader` and that it should run `jsc` to compile our `ecmascript`. We could also use TypeScript here if we were using that language in our project.

Next, we add our plugin. As you can see, this is in the `experimental` part of the config. Within `experimental`, we can define `plugins` and then simply add a tuple with the filesystem path to the Wasm file we previously built in our Rust plugin.

Here, we could also just add an existing plugin from npm. For example, we could use [loadable-components SWC plugin from npm](https://www.npmjs.com/package/@swc/plugin-loadable-components) instead of the path to our Wasm file.

We also add a basic `index.html` file:

```html
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8" />
        <title>webpack Plugin Test</title>
      </head>
      <body>
        <script src="./src/index.js"></script>
      </body>
    </html>
```

That’s it!

### Testing

Now let’s use the following command to run webpack:

```bash
    ./node_modules/.bin/webpack
```

You should see output similar to this:

```bash
    asset bundle.js 113 bytes \[compared for emit\] [minimized] (name: main)
    ./src/index.js 117 bytes \[built\] [code generated]
    swc-plugin-test (webpack 5.77.0) compiled successfully in 290 ms
```

Now, let’s check the output in `build/bundle.js`, which is our `index.js` file compiled with our plugin:

```bash
    console.debug("hello, world"),console.debug("this is already debuged"),console.debug("log"),console.trace("hello");
```

All `console.log` statements were replaced with `console.debug`, but the `console.trace` was not touched. Our plugin was called and it worked — success! 

## Conclusion

In this article, we explored how to build custom SWC plugins using Rust and then use them in webpack-based JavaScript or TypeScript projects to improve performance by speeding up our builds.

The plugin system described in this tutorial is still experimental and will likely change and mature. However, at the time of writing it already works quite smoothly. It’s already possible to port complex Babel plugins to SWC by replicating the AST-traversal and modification logic and porting it to Rust.

If you’re interested in diving deeper into SWC plugin development, you can also check out the [SWC Playground](https://play.swc.rs/) and the [SWC Viewer](https://github.com/IvanRodriCalleja/swc-viewer) project. These are both quite helpful resources for building SWC plugins.

SWC is an exciting project that was sorely needed in the bloated space of frontend build pipelines, providing the possibility to rebuild and rethink some of the existing machinery, while improving performance by orders of magnitude.

The examples used in this tutorial are available on [GitHub](https://github.com/zupzup/rust-swc-webpack-plugin).

