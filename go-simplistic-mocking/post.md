I think most people would agree that testing is an important part of developing software. However, writing tests is not always pleasant. Some things, like pure functions, are heaven to test and we should strive to write easily testable code whenever possible. 

For the sometimes-unavoidable cases where we have to test code which depends on external packages or simply on things which won't work well in an isolated unit-test, mocks can be of great help.

These "test-doubles" enable us to unit-test code which, for example, would try to connect to a database or make an HTTP call to the outside world, in isolation by pretending that said action took place and asserting that the rest of our assumptions hold. 

Of course, this isn't a perfect solution and as mentioned above, should be avoided if possible, but sometimes we won't be able to. In any case, it's usually a good idea to use interfaces in situations where dependencies are injected, because otherwise it's going to be hard to replace the malicious dependency with a mock in our tests. This also has the additional benefit, that we can swap out the concrete implementation of this dependency with another one which satisfies the same interface later on if we wish.

Now in Go (as in many other languages), there are several mocking frameworks available like [gomock](https://github.com/golang/mock), [testify](https://github.com/stretchr/testify) and some others. They work pretty well and usually (due to the current limitations of go's reflection capabilities) involve generating stubs for the interfaces to mock. 

As I have worked with many different mocking frameworks in different languages over the years, I wanted to try something new and write a very simplistic mocking utility for a small application I was testing, so that I wouldn't need to add another dependency just for tests. This little utility is described in the next few paragraphs - it is quite limited and won't generalize well to many, more advanced problems. 

Usually mocking frameworks also include ways to assert which parameters were passed into stubbed function calls, as well as many other fancy things. The goal here was simply to create a mechanism to stub out methods of an interface and to have a nice way of declaring expectations for what this interfaces methods should return for subsequent calls, to be able to test multiple success and failure cases without having to write several fake versions of the same method.

In the following example, there is an interface called `DataSource`, which has a method called `GetProducts`.

```go
type DataSource interface {
    GetProducts() ([]*model.Product, error)
}

type ProductFetcher struct {
    DataSource DataSource
}

func (f *ProductFetcher) Execute() (int, error) {
    ...fetch and count products...
}
```

There is also a `ProductFetcher` type, which has an `Execute` method. This `ProductFetcher` has a `DataSource`, which is used to fetch data from a database. Unfortunately, the concrete implementation of this `DataSource` actually connects to a database, which can't be expected to exist during test runs, so we need to mock it out.

We would like to be able to write tests similar to the following:

```go
func TestFetchProductsSuccess(t *testing.T) {
    var prods []*model.Product
    prods = append(prods, &model.Product{Code: "123"})
    prods = append(prods, &model.Product{Code: "234"})

    exps := make(Expectations)
    exps.Add("GetProducts", nil, prods)

    fetcher := ProductFetcher{
        DataSource: &mock.DataSource{Expectations: exps},
    }
    numProducts, err := fetcher.Execute()
    if numProducts != 2 || err != nil {
        t.Errorf("Error, actual: %v expected: %v", numProducts, 2)
        return
    }
}

func TestFetchProductsFailure(t *testing.T) {
    var prods []*model.Product

    exps := make(Expectations)
    exps.Add("GetProducts", prods, errors.New("err"))

    c := ExportCommand{
        DataSource: &mock.DataSource{Expectations: exps},
    }
    _, err := c.Execute()
    expected := "err"
    if err == nil || err.Error() != expected {
        t.Errorf("Error, actual: %v expected: %v", err.Error(), expected)
        return
    }
}
```

In these tests, we make the assumption that in the first case, the `DataSource` successfully returns two products and in the second case returns an error. We do this by telling our mock what it should return for the `GetProducts` method using `exps.Add`.

For this purpose, we create an `Expectations` type, which is a mapping of function names to `Expectation`:

```go
type Expectations map[string]*Expectation
type Expectation struct {
    CallCount     int
    DefaultReturn interface{}
    ReturnValues  []interface{}
}
```

This way, we can map a function name to desired return values. It's also possible to add multiple return values per function, in case a method gets called several times and there is also the `DefaultReturn` value, which is used in case an error should be returned and the other return value is not nillable.

Then we need a way to add these expectations to the mock object:

```go
func (e Expectations) Add(fn string, def interface{}, retValues ...interface{}) {
    exp := e[fn]
    if exp == nil {
        exp = &Expectation{}
    }
    exp.DefaultReturn = def
    exp.ReturnValues = append(exp.ReturnValues, retValues...)
    e[fn] = exp
}
```

Basically, we just add the given return values to `fn`'s map entry (which is created, if it doesn't exist yet).

The whole point of this exercise is that we don't have to stub out the interface methods for each different pair of return values, so we need a way for these stubs to access the `Expectations` for the current call:

```go
func (e Expectations) Return(fn string) (interface{}, error) {
    return e[fn].Return()
}

func (e *Expectation) Return() (interface{}, error) {
    res := e.ReturnValues[e.CallCount]
    var err error
    errTyped, ok := res.(error)
    if ok {
        err = errTyped
        res = e.DefaultReturn
    }
    e.CallCount = e.CallCount + 1
    return res, err
}
```

As you can see, this `Return` method only works for methods with 1 or 2 return values, but this could simply be generalized if we needed it, by either adding another `Return3` version, or by using an array of return values.

Other than that, we just fetch the provided return values for the current `CallCount`, increase said count and handle the case where we have an error and need to provide a sensible null-value for the other return value.

Now all we need to do is create the interface stubs. Because of the nifty little `Expectations` utility, we only need to create one stub, which we can then re-use for different calls:

```go
type DataSource struct {
    Expectations Expectations
}

func (d *DataSource) GetProducts() ([]*model.Product, error) {
    v, err := d.Expectations.Return("GetProducts")
    return v.([]*model.Product), err
}
```

The `GetProducts` method simply calls the `Expectations'` `Return` method, makes a type-assertion on the product array and returns everything.

Done!

Now we could re-use this utility for our other interfaces as well - we just need to create these two-liner stubs for all interface methods and we're good to go.

## Conclusion

Granted, this little thing won't be of much help when tackling a huge codebase with every imaginable edge-case there is, but for small projects like the one I built it for it might just be powerful enough and easily extensible due to its simplicity. 

In terms of extension, it would be trivial to add support for checking input parameters of stubbed method calls or to enable a  variable number of return values. 

More advanced features involving concurrency or the use of `reflect` might also be feasible to do yourself, but there comes a point where using one of the battle-tested frameworks like [testify](https://github.com/stretchr/testify) will be more efficient. 

This was a fun little exercise in my opinion, which *demystified* the magic behind mocking frameworks I've used for quite a long time for me, which is something I find both enjoyable and important.

There is no magic - just things you don't (fully) understand yet. :)

#### Resources

* [gomock](https://github.com/golang/mock)
* [testify](https://github.com/stretchr/testify)

