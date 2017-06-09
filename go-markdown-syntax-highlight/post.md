When I wrote the markdown-based [static blog generator](https://github.com/zupzup/blog-generator) for this blog, I used [highlightJS](https://highlightjs.org/) for syntax-highlighting, because it was something I had used before and seemed like a simple enough solution.

However, highlight is now the only piece of JavaScript code on this blog and I asked myself, if I couldn't just do the syntax-highlighting during page generation and remove the last bit of JS from the site to reduce its size by another few kilobytes.

While this might seem like overkill, I believe making the web simpler and easier to access for a wide audience has many benefits, especially for a blog.

I used Go for my blog-generator, so I looked for some existing syntax-highlighting packages and realized that there really isn't much available in this area (yet?).
Even [Hugo](https://gohugo.io/extras/highlighting/) uses the (absolutely amazing) python-based [pygments](http://pygments.org/) or external JavaScript like highlight or [google-prettify](https://github.com/google/code-prettify).

I wanted a native Go solution, without any external dependencies to python or any other language, as that always makes it harder to use and share a tool. There is a Go wrapper for pygments, but that doesn't get rid of the dependency.

So, what options do we have?

One solution I thought about was porting pygments to Go using [grumpy](https://github.com/google/grumpy), but I have zero experience with it and it seems very early in development for moving over a big, old project like pygments.

Another way to go would be to use a Go-based JavaScript interpreter like [otto](https://github.com/robertkrimen/otto) and just let highlight or google-prettify do its thing right there in the Go binary. While I think, this would work, I don't think it's a particularly pretty solution.

Also, the little experience I had so far with using JS in non-JS environments showed me, that this also won't "just work" out of the box.

Luckily, I found sourcegraph's great [syntaxhighlight](https://github.com/sourcegraph/syntaxhighlight) package, which is a native Go library for highlighting code, with an immensely simple API.
The package uses google-prettify classes in its output, so it is possible to use existing templates to style it.

At the moment, syntaxhighlight supports only JavaScript, Java, Ruby, Python, Go, and C, which is plenty for me, but of course can't compete with pygment's range of languages.

With all that out of the way, let's look at proof of concept I built using:

* [syntaxhighlight](https://github.com/sourcegraph/syntaxhighlight) - for syntax highlighting
* [blackfriday](https://github.com/russross/blackfriday) - for markdown-to-html conversion
* [google-prettify](https://github.com/google/code-prettify) - for templating (only CSS) 


## Implementation 

The basic idea is simple. We load a `markdown` file, convert it to `HTML`, find the code-parts, highlight them, replace them with the highlighted parts and render it to some template which includes the google-prettify CSS.

So, the template is the easiest part, and could look something like this:

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en-us" lang="en-us">
    <head>
        <title>Syntax Highlighting from Markdown with Go</title>
        <style>
            ...google prettify CSS...
        </style>
    </head>
    <body>
        {{.Content}}
    </body>
</html>
```

The next step is loading the file. In this example I just used `ioutil` to read the file:

```go
func main() {
    // load markdown file
    mdFile, err := ioutil.ReadFile("./example.md")
    if err != nil {
        log.Fatal(err)
    }
```

Then convert the markdown to HTML using `blackfriday`:

```go
    // convert markdown to html
    html := blackfriday.MarkdownCommon(mdFile)
```

Alright. At this point we have the `HTML` output for the given markdown file with some code-blocks, which are formatted like this:

```html
<pre><code class="language-go">
    ...some Code...
</code></pre>
```

The next step is to find these parts, take the code, highlight it and replace it back in again. Unfortunately, we [shouldn't use RegEx to parse HTML](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags), as the markup, for example, could contain the same tags we are looking for.

Another solution is to use Go's `html` package for parsing, or as I did in this case, to use [goquery](https://github.com/PuerkitoBio/goquery), which is a jQuery-like package implemented in Go.

There are many ways to achieve the task of parsing and replacing the code markup, so just pick your poison!

Using goquery, it could look like this:

```go
func replaceCodeParts(mdFile []byte) (string, error) {
    doc, err := goquery.NewDocumentFromReader(bytes.NewReader(mdFile))
    if err != nil {
        return "", err
    }

    // find code-parts via css selector and replace them with highlighted versions
    doc.Find("code[class*=\"language-\"]").Each(func(i int, s *goquery.Selection) {
        oldCode := s.Text()
        formatted, err := syntaxhighlight.AsHTML([]byte(oldCode))
        if err != nil {
            log.Fatal(err)
        }
        s.SetHtml(string(formatted))
    })

    new, err := doc.Html()
    if err != nil {
        return "", err
    }

    // replace unnecessarily added html tags
    new = strings.Replace(new, "<html><head></head><body>", "", 1)
    new = strings.Replace(new, "</body></html>", "", 1)
    return new, nil
}
```

First, we create a new `goquery Document`. The goquery API makes it easy for us to find the code-blocks using CSS-selectors like `code[class*=\"language-\"]`. 

Within this selector, we select the `Text` of the node, format it using syntaxhighlight's `AsHTML` method and set the resulting `HTML` on the node.

This way, the goquery `Document` is a valid HTML representation of the converted, syntax-highlighted markdown file. We get a string representation of this `Document` with `doc.Html()`.

At the end of the snippet, we remove the `<html>`, `<head>` and `<body>` tags goquery adds, as we won't need them in our template.

Alright. Now we just need to call the `replaceCodeParts` function and render the template to `os.Stdout`:

```go
    // replace code-parts with syntax-highlighted parts
    replaced, err := replaceCodeParts(html)
    if err != nil {
        log.Fatal(err)
    }

    // create template and write to stdout
    t, err := template.ParseFiles("./template.html")
    if err != nil {
        log.Fatal(err)
    }
    err = t.Execute(os.Stdout, struct{ Content string }{Content: replaced})
    if err != nil {
        log.Fatal(err)
    }
}
```

That's it. You can find the full code [here](https://github.com/zupzup/markdown-code-highlight-go).

## Conclusion 

There are many different ways to skin the syntax-highlighting-cat. This post showed a simple solution which uses only Go without any outside dependencies.

This approach doesn't come close to the power of [pygments](http://pygments.org/) of course, but for use-cases where external dependencies would be a burden, it could be a suitable option. 

I will try to implement the proof of concept shown in this post for this blog, to reduce its file-size and complexity even further, following a mindset I wish more people in the industry would start adopting.

Have fun highlightin'! :)

#### Resources

* [Full Example Code on Github](https://github.com/zupzup/markdown-code-highlight-go)
* [blackfriday](https://github.com/russross/blackfriday) 
* [syntaxhighlight](https://github.com/sourcegraph/syntaxhighlight) 
* [google-prettify](https://github.com/google/code-prettify) 
* [highlightjs](https://highlightjs.org/) 
* [otto](https://github.com/robertkrimen/otto)
* [pygments](http://pygments.org/) 
* [grumpy](https://github.com/google/grumpy)
* [Hugo](https://gohugo.io/)
