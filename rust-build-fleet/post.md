*This post was originally posted on the [LogRocket](https://blog.logrocket.com/introducing-fleet-improving-rusts-cargo/) blog on 21.07.2022 and was cross-posted here by the author.*

According to the [2021 Rust Survey](https://blog.rust-lang.org/2022/02/15/Rust-Survey-2021.html#challenges-ahead), Rust’s long compile time is still a big concern and an area for further improvement.

Especially when it comes to large projects or crates with many dependencies, Rust’s focus on run-time performance vs. compile-time performance becomes rather punishing, even in debug builds. This leads to a deterioration of developer experience and is a reason why some developers are still not willing to try Rust.

A lot of progress has been made and still is being made, some of which is documented in Nicholas Nethercote’s fantastic [blog posts](https://nnethercote.github.io/2022/02/25/how-to-speed-up-the-rust-compiler-in-2022.html) on the topic. He also wrote a chapter on compile time improvements in [The Rust Performance Book](https://nnethercote.github.io/perf-book/compile-times.html). Some of these techniques are actually used in Fleet to improve build times. If you’re interested in a deeper dive on the reasons behind why Rust has comparatively slow compile times, I suggest you read Brian Anderson’s fantastic [article](https://en.pingcap.com/blog/rust-compilation-model-calamity/) about the topic.

In any case, Rust’s build times will continue to be on the slower side for the foreseeable future, especially for larger projects. While there are some tweaks one can make to improve this situation, setting them up and keeping up-to-date on new developments, like flags and configuration options for improved build times, is cumbersome.

In this article, we’ll take a look at [Fleet.rs](https://fleet.rs/), which is essentially a one-tool-solution for improving your Rust build times, both for local development and for CI/CD-pipelines.

## What is Fleet?

The focus of our project is ease of use. Fleet doesn’t necessarily aim to reinvent the wheel and completely overhaul or restructure the way Rust builds work, but rather it wraps the existing build tools, tweaking optimizations together in a nice, configurable, intuitive tool that takes care of speeding up your builds. It works on Linux, Windows, and Mac OS.

Unfortunately, at this time, [Fleet is still in beta](https://github.com/dimensionhq/fleet) and only supports nightly `rustc`, but it’s being actively developed. Moving it to the `stable` toolchain is on the short list of upcoming improvements. 

That said, if you don’t feel comfortable using Fleet right away, or your current project setups are unable to use Fleet, there’s some good news. You can do most of the optimizations manually. Later in this article, we’ll go over them quickly, sharing some resources where you can learn more about them.

But first, let’s start by checking out how to install Fleet and how to use it in a project!

## Installing and using Fleet

To install Fleet, you’ll need Rust installed on your machine. If that’s taken care of, simply open your terminal and execute the correct, respective installation script:

For Linux:

```bash
    curl -L get.fleet.rs | sh
```

For Windows:

```bash
    iwr -useb windows.fleet.rs | iex
```

Once that’s done, you can set up  Fleet with one of four command line arguments:

* `-h /` `--help`: Print help information
* `-V, /` `--version`: Print version information
* `build`: Build a Fleet project
* `run`: Run a Fleet project

You can check out the additional, optional arguments for `run` and `build` in the Fleet [docs](https://fleet.rs/docs/commands/run). These are [somewhat similar to](https://blog.logrocket.com/demystifying-cargo-in-rust/) [C](https://blog.logrocket.com/demystifying-cargo-in-rust/)[argo, but it’s not a 1:1 replacement](https://blog.logrocket.com/demystifying-cargo-in-rust/), so be sure to check out the different configuration options if you have particular needs in terms of building your project.

If you plan to benchmark build times with and without Fleet, be sure to run clean builds and keep caching and preloading in mind when comparing the time it takes to build.

While Fleet claims to be [up to five times faster than Cargo](https://rustrepo.com/repo/dimensionhq-fleet)] on some builds, how big the actual performance gains are for your project in terms of compilation speed will depend on many different factors. For example, both your hardware, SSD vs. WSL (Windows System for Linux), and the code you’re trying to compile plus its dependencies.

In any case, if you currently feel that your project builds very slowly, install Fleet and give it a try to see if it improves the situation. Fleet takes nearly no time in terms of setup.

In addition to local development improvements, another important goal of Fleet is to improve CI/CD-pipelines. If you’re interested in trying out Fleet for your automated builds, be sure to check out their docs on setting it up with GitHub for Linux and Windows [here](https://fleet.rs/docs/ci/linux).

## Optimizations

At the time of writing this article, Fleet focuses on four different optimizations: using Ramdisk, optimizing the build through settings, using Sccache, and using a custom linker. 

You can find a short description in [this](https://github.com/dimensionhq/fleet/issues/34) ticket, but it’s likely that this list will change over time, especially when Fleet moves to `stable` and is developed further.

Let’s go over the different optimizations one-by-one and see what they actually do. The following will not be an extensive description, but rather a superficial overview of the different techniques with some tips and resources on how to use them. At the end of this section, there is also a link to a fantastic article describing how to manually improve compile times in Rust as well.

### Ramdisk

A Ramdisk, or Ramdrive, is essentially just a block of RAM that’s being used as if it were a hard disk to improve speed and in some cases to put less stress on hard disks.

The idea of this optimization is to put the `/target` folder of your build onto a Ramdisk to speed up the build. If you already have an SSD, this will only marginally improve build times, if even that. 

But, if you use WSL (Windows Subsystem for Linux) or a non-SSD harddisk, Ramdisk has the potential to massively improve performance.

There are plenty of tutorials on how to create Ramdisks for the different operating systems, but as a starting point, you can use the following two posts on how to do it on [Mac OS](https://fy.blackhats.net.au/blog/html/2019/08/26/using_ramdisks_with_cargo.html) and on [Linux](https://linuxhint.com/create-ramdisk-linux/).

### Build configuration

Fleet manipulates the build configuration by using compiler options and flags to boost performance as well.

One example for this is increasing `codegen-units`. This essentially increases parallelism in LLVM when it comes to compiling your code, but comes at the potential cost of runtime performance. This is usually not an issue for debug builds, where developer experience and hence faster builds are important, but definitely for release builds. You can read more about this flag [here](https://doc.rust-lang.org/stable/rustc/codegen-options/index.html#codegen-units).

Setting `codegen-units` manually is rather easy, just add it to the `rustflags` in your `~/.cargo/config.toml`:

```toml
    [target.x86_64-unknown-linux-gnu]
    rustflags = ["-C", "codegen-units=256"]
```

However, as mentioned above, you should definitely override this back to `1` for release builds.

Another option is to lower the optimization level for your debug builds. This means that the run-time performance will suffer, but the compiler has less work to do, which is usually what you want for iterating on your code-base. However, there might be exceptions to this. You can read more about [optimization levels](https://doc.rust-lang.org/stable/rustc/codegen-options/index.html#opt-level) [in the docs](https://doc.rust-lang.org/stable/rustc/codegen-options/index.html#opt-level). 

To set the optimization level to the lowest possible setting, add the code below to your `~/.cargo/config.toml`:

```toml
    [target.x86_64-unknown-linux-gnu]
    rustflags = ["-C", "opt-level=0"]
```

Again, be sure to only set this for debug builds and not for release builds. You wouldn’t want to have entirely unoptimized code in your production binary!

For lower optimization levels, as mentioned, you can try adding the `share-generics` flag, which enables the sharing of generics between multiple crates in your project, potentially saving the compiler from doing duplicate work.

For example, for Linux, you could add this to your `~/.cargo/config.toml`:


```toml
    [target.x86_64-unknown-linux-gnu]
    rustflags = ["-Z", "share-generics=y"]
```

### Sccache

The next optimization is using Mozillas [Sccache](https://github.com/mozilla/sccache). Sccache is a compiler-caching tool, which means it attempts to cache compilation results, i.e., across projects or crates and stores them on a disk, either locally, or in cloud storage.

This is particularly useful if you have several projects with many and sometimes large dependencies. Caching the results of compiling these different projects can prevent the compiler from duplicating work.

Especially in the context of CI/CD-pipelines, where builds are usually executed in the context of a freshly spawned instance or container without any locally existing cache, cloud-backed sccache can drastically improve build times. As every time a build runs, the cache is updated and can be reused by subsequent builds.

Fleet seamlessly introduces sccache into its builds, but doing this manually is not particularly difficult either. Simply follow the instructions for [installation and usage](https://github.com/mozilla/sccache#usage) for sccache.

### Custom Linker

Finally, Fleet also configures and uses a custom linker, to improve build performance. Especially for large projects with deep dependency trees, the compiler spends a lot of time linking. In such cases, using the fastest possible linker can greatly improve compilation times.

The list below includes the correct linker to use for each operating system: 


- Linux: `clang + lld` (potentially soon [mold](https://github.com/rui314/mold))
- Windows: `rust-lld.exe`
- Mac OS: `zld`

Configuring a custom linker is not particularly difficult. Essentially, it boils down to installing the linker and then configuring Cargo to use it. For example, using `zld` on Mac OS can be implemented by adding the following config to your `~/.cargo/config`:

```toml
    [target.x86_64-apple-darwin]
    rustflags = ["-C", "link-arg=-fuse-ld=<path to zld>"]
```

On Linux, `lld`, or `mold`, is the best choices for Rust. Fleet doesn’t use `mold` yet due to [license issues](https://github.com/dimensionhq/fleet/issues/13), but you can use it in your build locally by simply following the steps for Rust in the [mold docs](https://github.com/rui314/mold#how-to-use).

That’s it for the optimizations Fleet performs under the hood and how you can use them manually to try and improve your build times.

After this short overview, another fantastic resource for improving your build times, if you’re reluctant to use Fleet at this point, is Matthias Endlers [blog post](https://endler.dev/2020/rust-compile-times/) about the topic.

## Conclusion

Fleet has great potential, especially for people who do not enjoy fuzzing around with build pipelines, or build processes in general. It provides a powerful all-in-one package of multi-platform and multi-environment optimizations in terms of build speed, so it’s well worth a try if you’re struggling with build times.

Beyond that, we touched on some of the optimizations Fleet does in the background and those will, if you’re willing to put in a little time to figure out what they do and how to introduce them in your setup, help alleviating your compile-speed pain as well.

That said, often times the reason behind slow build times is, that a project depends on many and/or very large crates. Managing your dependencies well, with a minimalist mindset, introducing only the minimal version of whatever you need and in some cases, instead of adding an existing crate, building the needed functionality from scratch for your specific use-case, will not only keep your build times low, but also reduce complexity and increase the maintainability of your code.

Happy Rust’ing! :)

