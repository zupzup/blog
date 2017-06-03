TBD

## Example Implementation 

TBD

```go
func main() {
    // load markdown file
    mdFile, err := ioutil.ReadFile("./example.md")
    if err != nil {
        log.Fatal(err)
    }

    // convert markdown to html
    html := blackfriday.MarkdownCommon(mdFile)

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

That's it. You can find the full code [here](https://github.com/zupzup/markdown-code-highlight-go).

## Conclusion 



#### Resources

* [Full Example Code on Github](https://github.com/zupzup/markdown-code-highlight-go)
* [blackfriday](https://github.com/russross/blackfriday) 
* [syntaxhighlight](https://github.com/sourcegraph/syntaxhighlight) 
* [google-prettify](https://github.com/google/code-prettify) 
* [highlightjs](https://highlightjs.org/) 
* [otto](https://github.com/robertkrimen/otto)
* [pygments](http://pygments.org/) 
* [hugo highlithgin](https://gohugo.io/extras/highlighting/)
* [grumpy](https://github.com/google/grumpy)
