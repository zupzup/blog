In a [previous post](https://zupzup.org/go-ast-traversal/), I showed a basic example of how to traverse an AST with Go. The ability to traverse and analyse the AST of a program is useful for building code-analysis tools and such, but the real fun starts when we manipulate the Abstract Syntax Tree of a program, which enables us to build powerful developer tools.

In this post, we will create a simple tool that does something useful in regards to documentation. The tool will parse a given Go source file and, for every **exported** function without a **Doc string**, it will spit out a warning and create a `// TODO: document exported function`-placeholder comment where the Doc-string should be.

While this very simple tool probably won't change the world, it's small enough to showcase the general concept of how to parse to an AST, manipulate it and then write the changed code back out.

Let's get started.

## Code Example

First, let's look at our source file called `test.go`:

```go
func main() {
    fmt.Println("testprogram")
    DoStuff()
}

func unexportedFunction() {}

// Whatever does other stuff
func Whatever() {}

func AnExportedFunction() {}

func DoStuff() {}

// DoOtherStuff does other stuff
func DoOtherStuff() {}
```

Our first step is to parse this `test.go` file into an AST:

```go
// parse file
fset := token.NewFileSet()
node, err := parser.ParseFile(fset, "test.go", nil, parser.ParseComments)
if err != nil {
    log.Fatal(err)
}
```

Our strategy is to identify all exported functions without a Doc-string and add the `TODO` comment on top of them.
For this purpose, we also need to identify and collect all comments in the AST, to be able to position the new comments correctly in the file's `Comments` list.

To traverse the AST, we use the `ast.Inspect` function:

```go
comments := []*ast.CommentGroup{}
ast.Inspect(node, func(n ast.Node) bool {
    // collect comments
    c, ok := n.(*ast.CommentGroup)
    if ok {
        comments = append(comments, c)
    }
    // handle function declarations without documentation
    fn, ok := n.(*ast.FuncDecl)
    if ok {
        if fn.Name.IsExported() && fn.Doc.Text() == "" {
            // print warning
            fmt.Printf("exported function declaration without documentation
                found on line %d: \n\t%s\n", fset.Position(fn.Pos()).Line, fn.Name.Name)
        }
    }
})
```

First, we identify and collect all `ast.CommentGroup` nodes, which are the existing comments in the code.
We also identify `ast.FuncDecl` nodes, which are function declarations and, if they are exported and their `fn.Doc.Text()` - the Doc string - is empty, we print a warning with the position and name of the undocumented function.

After identifying our targets, we need to manipulate them by including our `TODO`-comment into their Doc-string. We do this by creating a new `ast.Comment` and `ast.CommentGroup` and set `fn.Doc` to that comment group:

```go
// create todo-comment
comment := &ast.Comment{
    Text:  "// TODO: document exported function",
    Slash: fn.Pos() - 1,
}
// create CommentGroup and set it to the function's documentation comment
cg := &ast.CommentGroup{
    List: []*ast.Comment{comment},
}
fn.Doc = cg
```

What's important here is, that we need to set the `Slash` property of the newly created `ast.Comment` node to `fn.Pos() - 1`, to properly position the comment. This is the identified function's position in the file, expressed as its offset and we reduce it by 1 to get to the line just above the function.

After that's done, we set the `node`'s `Comments` to our collected `CommentGroup` list, so they are rendered and write the AST, pretty-printed with the `go/printer` package, to a file called `new.go`:

```go
// set ast's comments to the collected comments
node.Comments = comments
// write changed AST to file
f, err := os.Create("new.go")
defer f.Close()
if err := printer.Fprint(f, fset, node); err != nil {
    log.Fatal(err)
}
```

The result in `new.go` looks like this, just like we planned:

```go
func main() {
    fmt.Println("testprogram")
    DoStuff()
}

func unexportedFunction() {}

// Whatever does other stuff
func Whatever() {}

// TODO: document exported function
func AnExportedFunction() {}

// TODO: document exported function
func DoStuff() {}

// DoOtherStuff does other stuff
func DoOtherStuff() {}
```

That's it. :)

The full example code can be found [here](https://github.com/zupzup/ast-manipulation-example)

## Conclusion

Due to the complexity of Software Engineering, good developer tools are essential for high developer efficiency. Although Go already has a great ecosystem of powerful tools, the ability to build your own tools opens some interesting possibilities, especially when it comes to tools tailored explicitly to your own unique needs.

Go provides good, well documented libraries in the standard library to write such custom-tailored tools. Additionally, the knowledge and skills necessary to build such tools, in my opinion, also improve one's understanding of the language itself.

Another benefit of writing your own tools is that it's just a great feeling to use your own, hand-crafted tool to boost your development flow. :)

#### Resources

* [AST-Traversal in Go](https://zupzup.org/go-ast-traversal/)
* [Code Example](https://github.com/zupzup/ast-manipulation-example)
* [Go token package](https://golang.org/pkg/go/token/)
* [Go parser package](https://golang.org/pkg/go/parser/)
* [Go ast package](https://golang.org/pkg/go/ast/)
* [Go printer package](https://golang.org/pkg/go/printer/)

