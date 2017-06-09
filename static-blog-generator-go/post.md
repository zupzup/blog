A static-site-generator is a tool which, given some input (e.g. markdown) generates a fully static website using HTML, CSS and JavaScript.

Why is this cool? Well, for one it's a lot easier to host a static site and it's also (usually) quite a bit faster and resource-friendly.

Static sites aren't the best choice for all use-cases, but for mostly non-interactive websites such as blogs, they are great.

In this post I will describe the [static blog generator](https://github.com/zupzup/blog-generator) I wrote in Go, which powers this blog.

## Motivation 

You might be familiar with static-site-generators like the great [Hugo](https://gohugo.io), which has just about all the features one could hope for in regards to static site generation.

So why would I write another tool just like it, with fewer capabilities (which probably doesn't work as well)? The reason was twofold.

One reason was, that I wanted to dive deeper into Go and a command-line-based static-site-generator seemed like a great playground to hone my skills.

The second reason was simply that I had never done it before. I've done my fair share of web-development in all shapes and forms, but I never created a tool like this.

This made it intriguing because I theoretically had all the prerequisites and skills to build such a tool with my background in web-development, but I had never tried doing it.

I was hooked, implemented it within about 2 weeks and had a blast doing it. I used my blog-generator for creating this blog since it started and it worked great so far. :)

## Concept

Early on, I decided to write my blog posts in `Markdown` and keep them in a [GitHub Repo](https://github.com/zupzup/blog). The posts are structured in folders, which represent the url of the blog post.

For metadata such as publication date, tags, title and subtitle I decided on keeping a `meta.yml` file with each `post.md` file with the following format:

```yml
title: Playing Around with BoltDB 
short: "Looking for a simple key/value store for your Go applications? Look no further!"
date: 20.04.2017
tags:
    - golang
    - go
    - boltdb
    - bolt
```

This allowed me to seperate the content from the metadata, but still keep everything in the same place, where I'd find it later.

The GitHub Repo was my data source. The next step was to think about the features I wanted. I decided on the following:

* Very Lean (Landingpage should be 1 Request < 10K gzipped)
* Listing for the Landingpage and an Archive
* Possibility to use syntax-highlighted Code and Images within Blog Posts
* Tags
* RSS feed (index.xml)
* Optional Static Pages (e.g. About)
* High Maintainability - use the least amount of templates possible
* sitemap.xml for SEO
* Local preview of the whole blog (a simple `run.sh` script)

Quite a healthy feature set. What was very important for me from the start, was to keep everything simple, fast and clean - without any third-party trackers or ads, which compromise privacy and slow everything down.

Based on these ideas, I started making a rough plan of the architecture and started coding.

## Architectural Overview

The application is simple enough. The high-level elements are:

* `CLI`
* `DataSource`
* `Generators`

The `CLI` in this case is very simple, as I didn't add any features in terms of configurability. It basically just fetches data from the `DataSource` and runs the `Generators` on it.

The `DataSource` interface looks like this:

```go
type DataSource interface {
    Fetch(from, to string) ([]string, error)
}
```

The `Generator` interface looks like this:

```go
type Generator interface {
	Generate() error
}
```

Pretty simple. Each `Generator` also receives a configuration struct, which contains all the necessary data for generation.

There are 7 `Generators` at the time of writing this post:

* `SiteGenerator`
* `ListingGenerator`
* `PostGenerator`
* `RSSGenerator`
* `SitemapGenerator`
* `StaticsGenerator`
* `TagsGenerator`

Where the `SiteGenerator` is the meta-generator, which calls all other generators and outputs the whole static website.

The generation is based on `HTML` templates using Go's `html/template` package.

## Implementation Details

In this section I will just cover a few selected parts I think might be interesting, such as...TBD

```go
type Config struct {
    Birthday time.Time
    Height float
}
```

That's it. You can find the full code [here](https://github.com/zupzup/blog-generator).

## Conclusion 

Writing my [blog-generator](https://github.com/zupzup/blog-generator) was an absolute blast and a great learning experience. It's also quite satisfying to use my own, hand-crafted tool for creating and publishing these blog posts.

In order to publish my posts to AWS, I also create [static-aws-deploy](https://github.com/zupzup/static-aws-deploy), another Go command-line tool, which I covered in [this post](https://zupzup.org/static-deploy-tool/).

If you want to use the tool yourself, just fork the repo and change the configuration. However, I didn't put much time in customizability or configurability, as [Hugo](https://gohugo.io) provides all that and more.

Of course one should strive not to re-invent the wheel all the time, but sometimes re-inventing a wheel or two can be rewarding and can help you learn quite a bit in the process. :)

#### Resources

* [blog-generator on Github](https://github.com/zupzup/blog-generator)
* [Hugo](https://gohugo.io)
