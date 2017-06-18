A static-site-generator is a tool which, given some input (e.g. markdown) generates a fully static website using HTML, CSS and JavaScript.

Why is this cool? Well, for one it's a lot easier to host a static site and it's also (usually) quite a bit faster and resource-friendly. Static sites aren't the best choice for all use-cases, but for mostly non-interactive websites such as blogs they are great.

In this post, I will describe the [static blog generator](https://github.com/zupzup/blog-generator) I wrote in Go, which powers this blog.

## Motivation 

You might be familiar with static-site-generators like the great [Hugo](https://gohugo.io), which has just about all the features one could hope for in regards to static site generation.

So why would I write another tool just like it with fewer capabilities? The reason was twofold.

One reason was that I wanted to dive deeper into Go and a command-line-based static-site-generator seemed like a great playground to hone my skills.

The second reason was simply that I had never done it before. I've done my fair share of web-development, but I never created a static-site-generator.

This made it intriguing because I theoretically had all the prerequisites and skills to build such a tool with my background in web-development, but I had never tried doing it.

I was hooked, implemented it within about 2 weeks and had a great time doing it. I used my blog-generator for creating this blog and it worked great so far. :)

## Concept

Early on, I decided to write my blog posts in `markdown` and keep them in a [GitHub Repo](https://github.com/zupzup/blog). The posts are structured in folders, which represent the url of the blog post.

For metadata, such as publication date, tags, title and subtitle I decided on keeping a `meta.yml` file with each `post.md` file with the following format:

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

This allowed me to separate the content from the metadata, but still keep everything in the same place, where I'd find it later.

The GitHub Repo was my data source. The next step was to think about features and I came up with this List:

* Very Lean (landing page should be 1 Request < 10K gzipped)
* Listing for the landing page and an Archive
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

In this section I will just cover a few selected parts I think might be interesting, such as the git `DataSource` and the different `Generators`.

### DataSource

First up, we need some data to generate our blog from. This data, as mentioned above has the form of a git repository. The following `Fetch` function captures most of what the `DataSource` implementation does:

```go
func (ds *GitDataSource) Fetch(from, to string) ([]string, error) {
    fmt.Printf("Fetching data from %s into %s...\n", from, to)
    if err := createFolderIfNotExist(to); err != nil {
        return nil, err
    }
    if err := clearFolder(to); err != nil {
        return nil, err
    }
    if err := cloneRepo(to, from); err != nil {
        return nil, err
    }
    dirs, err := getContentFolders(to)
    if err != nil {
        return nil, err
    }
    fmt.Print("Fetching complete.\n")
    return dirs, nil
}
```

`Fetch` is called with two parameters `from`, which is a repository URL and `to`, which is the destination folder. The function creates and clears the destination folder, clones the repository using `os/exec` plus a git command and finally reads the folder, returning a list of paths for all the files within the repository.

As mentioned above, the repository contains only folders, which represent the different blog posts. The array with these folder paths is then passed to the generators, which can then do their thing for each of the blog posts within the repository.

### Kicking it all off

After the `Fetch` comes the `Generate` phase. When the `blog-generator` is executed, the following code is executed on the highest level:

```go
ds := datasource.New()
dirs, err := ds.Fetch(RepoURL, TmpFolder)
if err != nil {
    log.Fatal(err)
}
g := generator.New(&generator.SiteConfig{
    Sources:     dirs,
    Destination: DestFolder,
})
err = g.Generate()
if err != nil {
    log.Fatal(err)
}
```

The `generator.New` function creates a new `SiteGenerator` which is basically a generator, which calls other generators. It's passed a destination folder and the directories for the blog posts within the repository. 

As every `Generator` implementing the interface mentioned above, the `SiteGenerator` has a `Generate` method, which returns an error. The `Generate` method of the `SiteGenerator` prepares the destination folder, reads in templates, prepares data structures for the blog posts, registers the other generators and concurrently runs them.

The `SiteGenerator` also registers some settings for the blog like the URL, Language, Date Format etc. These settings are simply global constants, which is certainly not the prettiest solution or the most scalable, but it's simple and that was the highest goal here.

### Posts

The most important concept on a blog are - surprise, surprise - blog posts! In the context of this blog-generator, they are represented by the following data-structure:

```go
type Post struct {
    Name      string
    HTML      []byte
    Meta      *Meta
    ImagesDir string
    Images    []string
}
```

These posts are created by iterating over the folders in the repository, reading the `meta.yml` file, converting the `post.md` file to HTML and by adding images, if there are any.

Quite a bit of work, but once we have the posts represented as a data structure, the generation of posts is quite simple and looks like this:

```go
func (g *PostGenerator) Generate() error {
    post := g.Config.Post
    destination := g.Config.Destination
    t := g.Config.Template
    staticPath := fmt.Sprintf("%s%s", destination, post.Name)
    if err := os.Mkdir(staticPath, os.ModePerm); err != nil {
      return fmt.Errorf("error creating directory at %s: %v", staticPath, err)
    }
    if post.ImagesDir != "" {
      if err := copyImagesDir(post.ImagesDir, staticPath); err != nil {
          return err
      }
    }
    if err := writeIndexHTML(staticPath, post.Meta.Title, template.HTML(string(post.HTML)), t); err != nil {
      return err
    }
    return nil
}
```

First, we create a directory for the post, then we copy the images in there and finally create the post's index.html file using templating. The `PostGenerator` also implements syntax-highlighting, which I described in [this post](https://zupzup.org/go-markdown-syntax-highlight/).

### Listing Creation 

When a user comes to the landing page of the blog, she sees the latest posts with information like the reading time of the article and a short description. For this feature and for the archive, I implemented the `ListingGenerator`, which takes the following config:

```go
type ListingConfig struct {
    Posts                  []*Post
    Template               *template.Template
    Destination, PageTitle string
}
```

The `Generate` method of this generator iterates over the post, assembles their metadata and creates short blocks based on the given template. Then these blocks are appended and written to the index template.

I liked medium's feature to approximate the time to read an article, so I implemented my own version of it, based on the assumption that an average human reads about 200 words per minute. Images also count towards the overall reading time with a constant 12 seconds added for each `img` tag in the post. This will obviously not scale for arbitrary content, but should be a fine approximation for my usual articles:

```go
func calculateTimeToRead(input string) string {
    // an average human reads about 200 wpm
    var secondsPerWord = 60.0 / 200.0
    // multiply with the amount of words
    words := secondsPerWord * float64(len(strings.Split(input, " ")))
    // add 12 seconds for each image
    images := 12.0 * strings.Count(input, "<img")
    result := (words + float64(images)) / 60.0
    if result < 1.0 {
        result = 1.0
    }
    return fmt.Sprintf("%.0fm", result)
}
```

### Tags

Next, to have a way to categorize and filter the posts by topic, I opted to implement a simple tagging mechanism. Posts have a list of tags in their `meta.yml` file. These tags should be listed on a separate `Tags` Page and upon clicking on a tag, the user is supposed to see a listing of posts with the selected tag.

First up, we create a map from tag to `Post`: 

```go
func createTagPostsMap(posts []*Post) map[string][]*Post {
result := make(map[string][]*Post)
    for _, post := range posts {
        for _, tag := range post.Meta.Tags {
            key := strings.ToLower(tag)
             if result[key] == nil {
                 result[key] = []*Post{post}
             } else {
                 result[key] = append(result[key], post)
             }
        }
    }
    return result
}
```

Then, there are two tasks to implement:

* Tags Page
* List of Posts for a selected Tag

The data structure of a `Tag` looks like this:

```go
type Tag struct {
    Name  string
    Link  string
    Count int
}
```

So, we have the actual tag (Name), the Link to the tag's listing page and the amount of posts with this tag. These tags are created from the `tagPostsMap` and then sorted by `Count` descending:

```go
tags := []*Tag{}
for tag, posts := range tagPostsMap {
    tags = append(tags, &Tag{Name: tag, Link: getTagLink(tag), Count: len(posts)})
}
sort.Sort(ByCountDesc(tags))
```

The Tags Page basically just consists of this list rendered into the `tags/index.html` file.

The List of Posts for a selected Tag can be achieved using the `ListingGenerator` described above. We just need to iterate the tags, create a folder for each tag, select the posts to display and generate a listing for them.

### Sitemap & RSS

To improve searchability on the web, it's a good idea to have a `sitemap.xml` which can be crawled by bots. Creating such a file is fairly straightforward and can be done using the Go standard library.

In this tool, however, I opted to use the great [etree](https://github.com/beevik/etree) library, which provides a nice API for creating and reading XML.

The `SitemapGenerator` uses this config:

```go
type SitemapConfig struct {
    Posts       []*Post
    TagPostsMap map[string][]*Post
    Destination string
}
```

`blog-generator` takes a basic approach to the sitemap and just generates `url` and `image` locations by using the `addURL` function:

```go
func addURL(element *etree.Element, location string, images []string) {
    url := element.CreateElement("url")
     loc := url.CreateElement("loc")
     loc.SetText(fmt.Sprintf("%s/%s/", blogURL, location))

     if len(images) > 0 {
         for _, image := range images {
            img := url.CreateElement("image:image")
             imgLoc := img.CreateElement("image:loc")
             imgLoc.SetText(fmt.Sprintf("%s/%s/images/%s", blogURL, location, image))
         }
     }
}
```

After creating the XML document with `etree`, it's just saved to a file and stored in the output folder.

RSS generation works the same way - iterate all posts and create XML entries for each post, then write to `index.xml`.

### Handling Statics

The last concept I needed were entirely static assets like a `favicon.ico` or a static page like `About`. To do this, the tool runs the `StaticsGenerator` with this config:

```go
type StaticsConfig struct {
    FileToDestination map[string]string
    TemplateToFile    map[string]string
    Template          *template.Template
}
```

The `FileToDestination`-map represents static files like images or the `robots.txt` and `TemplateToFile` is a mapping from templates in the `static` folder to their designated output path.

This configuration could look like this in practice:

```go
fileToDestination := map[string]string{
    "static/favicon.ico": fmt.Sprintf("%s/favicon.ico", destination),
    "static/robots.txt":  fmt.Sprintf("%s/robots.txt", destination),
    "static/about.png":   fmt.Sprintf("%s/about.png", destination),
}
templateToFile := map[string]string{
    "static/about.html": fmt.Sprintf("%s/about/index.html", destination),
}
statg := StaticsGenerator{&StaticsConfig{
FileToDestination: fileToDestination,
   TemplateToFile:    templateToFile,
   Template:          t,
}}
```

The code for generating these statics is not particularly interesting - as you can imagine, the files are just iterated and copied to the given destination.

### Parallel Execution

For `blog-generator` to be fast, the generators are all run in parallel. For this purpose, they all follow the `Generator` interface - this way we can put them all inside a slice and concurrently call `Generate` for all of them.

The generators all work independently of one another and don't use any global state mutation, so parallelizing them was a simple exercise of using channels and a `sync.WaitGroup` like this:

```go
func runTasks(posts []*Post, t *template.Template, destination string) error {
    var wg sync.WaitGroup
    finished := make(chan bool, 1)
    errors := make(chan error, 1)
    pool := make(chan struct{}, 50)
    generators := []Generator{}

    for _, post := range posts {
        pg := PostGenerator{&PostConfig{
            Post:        post,
             Destination: destination,
             Template:    t,
        }}
        generators = append(generators, &pg)
    }

    fg := ListingGenerator{&ListingConfig{
        Posts:       posts[:getNumOfPagesOnFrontpage(posts)],
        Template:    t,
        Destination: destination,
        PageTitle:   "",
    }}

    ...create the other generators...

    generators = append(generators, &fg, &ag, &tg, &sg, &rg, &statg)

    for _, generator := range generators {
        wg.Add(1)
        go func(g Generator) {
            defer wg.Done()
            pool <- struct{}{}
            defer func() { <-pool }()
            if err := g.Generate(); err != nil {
                errors <- err
            }
        }(generator)
    }

    go func() {
        wg.Wait()
        close(finished)
    }()

    select {
    case <-finished:
        return nil
    case err := <-errors:
        if err != nil {
           return err
        }
    }
    return nil
}
```

The `runTasks` function uses a pool of max. 50 goroutines, creates all generators, adds them to a slice and then runs them in parallel.

These examples were just a short dive into the basic concepts used to write a static-site generator in Go.

If you're interested in the full implementation, you can find the code [here](https://github.com/zupzup/blog-generator).

## Conclusion 

Writing my [blog-generator](https://github.com/zupzup/blog-generator) was an absolute blast and a great learning experience. It's also quite satisfying to use my own hand-crafted tool for creating my blog.

To publish my posts to AWS, I also created [static-aws-deploy](https://github.com/zupzup/static-aws-deploy), another Go command-line tool, which I covered in [this post](https://zupzup.org/static-deploy-tool/).

If you want to use the tool yourself, just fork the repo and change the configuration. However, I didn't put much time into customizability or configurability, as [Hugo](https://gohugo.io) provides all that and more.

Of course, one should strive not to re-invent the wheel all the time, but sometimes re-inventing a wheel or two can be rewarding and can help you learn quite a bit in the process. :)

#### Resources

* [blog-generator on GitHub](https://github.com/zupzup/blog-generator)
* [Hugo](https://gohugo.io)
* [etree](https://github.com/beevik/etree)
