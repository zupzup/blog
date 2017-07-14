Inspired by Fatih Arslan's great [talk](https://www.youtube.com/watch?v=oxc8B2fjDvY) about writing tools for Go as well as by Gary Bernhardt's [screencasts](https://www.destroyallsoftware.com/screencasts) on computation, I decided to try myself at some AST-traversal using Go in preparation of diving a bit more into the subject.

This short post will just introduce the absolute basics of how to:

* Parse a Go program into an Abstract Syntax Tree
* Analyse the AST using only the standard library 

In the future, I plan to cover more advanced topics in this area as well.

Let's get started.

## Code Example

First up, let's take a look at the file we want to parse:

```go
package main

import "fmt"
import "strings"

func main() {
    hello := "Hello"
    world := "World"
    words := []string{hello, world}
    SayHello(words)
}

// SayHello says Hello
func SayHello(words []string) {
    fmt.Println(joinStrings(words))
}

// joinStrings joins strings
func joinStrings(words []string) string {
    return strings.Join(words, ", ")
}
```

Nothing fancy - literally an overly complex `Hello, World` program. Even this small bit of code, however, contains many interesting elements like functions, variables, comments, imports, an export and function calls, which is enough to get our feet wet.

The next step is to parse the file into an AST. For this purpose, we use the `go/parser` package like this:

```go
fset := token.NewFileSet()
node, err := parser.ParseFile(fset, "test.go", nil, parser.ParseComments)
if err != nil {
    log.Fatal(err)
}
```

The first line uses the `go/token` package to create a new FileSet, which represents a set of source files and which we need for the parser. 

Then we simply call `parser.ParseFile` with the mode `parser.ParseComments`, which parses everything including comments and we get a new `ast.File` as the return value. This `ast.File` is a representation of a Go source file and looks like this (from the official [docs](https://golang.org/pkg/go/ast/#File)):

```go
type File struct {
    Doc        *CommentGroup   // associated documentation; or nil
    Package    token.Pos       // position of "package" keyword
    Name       *Ident          // package name
    Decls      []Decl          // top-level declarations; or nil
    Scope      *Scope          // package scope (this file only)
    Imports    []*ImportSpec   // imports in this file
    Unresolved []*Ident        // unresolved identifiers in this file
    Comments   []*CommentGroup // list of all comments in the source file
}
```

We can already use this data structure to start analysing our program. For example, we can list all of our imports using the `node.Imports` field:

```go
fmt.Println("Imports:")
for _, i := range node.Imports {
    fmt.Println(i.Path.Value)
}
```

Now, this by itself is not extremely interesting, but I think it's not hard to imagine the usefulness of even this simple example. We could for example, for a whole codebase, analyse all external dependencies, collected using just these few lines of code.

We can do the same with other parts of the `ast.File` data structure such as collecting comments, or functions: 

```go
fmt.Println("Comments:")
for _, c := range node.Comments {
    fmt.Print(c.Text())
}

fmt.Println("Functions:")
for _, f := range node.Decls {
    fn, ok := f.(*ast.FuncDecl)
    if !ok {
        continue
    }
    fmt.Println(fn.Name.Name)
}
```

Comments look the same as imports, but for functions we actually look at all the declarations (`node.Decls`) in the file and check if they are functions (`*ast.FuncDecl`). We could further inspect the body of the function or check for other types of declarations.

This is already pretty nice, but as you can imagine, traversing through the AST like this can be quite bothersome, so the `go/ast` package provides the `ast.Inspect` function, which makes this a lot nicer and easier for us: 

```go
func Inspect(node Node, f func(Node) bool)
```

The `Inspect` function takes a node (such as our `ast.File`) and a visitor function, which is called for every node encountered in the whole AST, which is walked depth-first.

Using this powerful tool, we can, for example, look for return statements in our code:

```go
ast.Inspect(node, func(n ast.Node) bool {
    // Find Return Statements
    ret, ok := n.(*ast.ReturnStmt)
    if ok {
        fmt.Printf("return statement found on line %d:\n\t", fset.Position(ret.Pos()).Line)
        printer.Fprint(os.Stdout, fset, ret)
        return true
    }
    return true
})
```

We simply make a type assertion for the type of node we are interested in and then do something with it. In this case, we simply get the line the statement is on using `fset.Position()` and print the actual line of code using the `go/printer` package's `printer.Fprint()` function.

Another use-case could be to again find all functions and find out whether they are exported or not:

```go
// Find Functions
fn, ok := n.(*ast.FuncDecl)
if ok {
    var exported string
    if fn.Name.IsExported() {
        exported = "exported "
    }
    fmt.Printf("%sfunction declaration found on line %d: \n\t%s\n", exported, fset.Position(fn.Pos()).Line, fn.Name.Name)
    return true
}
```

The structure is the same as above - we assert that the node is a function declaration and then use the type-asserted node to find out more about it. Pretty nice.

A gist with the full example code can be found [here](https://gist.github.com/zupzup/2c7d2baef1120933ef7b2f1360e2e5a0)

## Conclusion

This concludes our short excursion into the land of Basic AST traversal in Go. The short example above only scratches the surface of interacting with the AST, but showcases the nice API for tackling these kinds of problems within the Go standard library.

With APIs like these and the general simplicity of the language, it's no wonder there is such a rich ecosystem of tools for Go already.

Also, in my opinion, these techniques aren't only useful for building complex general-purpose tools, but can also be helpful for analysing your own codebase in a certain way or automating something very specific to your project.

The possibilities seem endless and it's quite a bit of fun to walk around code you've written and see what's what in an automated way! :)

#### Resources

* [Talk by Fatih Arslan at GothamGo](https://www.youtube.com/watch?v=oxc8B2fjDvY)
* [DestroyAllSoftware Screencasts](https://www.destroyallsoftware.com/screencasts)
* [Go token package](https://golang.org/pkg/go/token/)
* [Go parser package](https://golang.org/pkg/go/parser/)
* [Go ast package](https://golang.org/pkg/go/ast/)
* [Go printer package](https://golang.org/pkg/go/printer/)
* [Full Code Example](https://gist.github.com/zupzup/2c7d2baef1120933ef7b2f1360e2e5a0)

