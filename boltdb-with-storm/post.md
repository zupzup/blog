Last month I [wrote](https://zupzup.org/boltdb-example/) about BoltDB and showed an example for using it with a simple Data Model.

BoltDB handles everything with `[]byte` arrays, which is simple, straightforward and fast, but has some trade-offs in the usability department as it makes complex queries and nested structures a bit bothersome.

This time, we'll take a look at [storm](https://github.com/asdine/storm), which is a toolkit for BoltDB providing indexes, an advanced query system and many more utilities to tackle complex use-cases.

Of course, you might argue, that it makes sense to use another data store entirely, if the problems you're trying to solve don't map well to BoltDB and you would be right. However, in the real world we don't always know at the start of a project what we'll need later on and if we decided to use bolt but have a few cases where it's not optimal, it might be better to use an abstraction like storm instead of adding a second data store. 

In this post, we'll take a look at some of the API improvements `storm` provides for cases where we need a little bit more than key/value access.

Let's start with the Data Model we want to use.

## Data Model

We'll use the same Data Model again and see how storm can help us.

```go
type Config struct {
    ID int `storm:"id, increment"`
    Birthday time.Time
    Height float
}
```

Configuration - we always want to use the latest one.

```go
type Weight struct {
    ID int `storm:"id, increment"`
    Date time.Time
    Weight float64
}
```

Weight Timeline. The latest weight-entry is the current weight.

```go
type Entry struct {
    ID int `storm:"id, increment"`
    Date time.Time
    Food string
    Calories int
}
```

Entries, displayed by day. It should be possible to query by time-spans.

## Implementation

As you can see from the Data Model structs, we added a `storm:"id, increment"` tag to each model's ID. This, non-surprisingly, creates a self-incrementing PK for these models. 

Setting up the DB works basically the same way:

```go
db, err := storm.Open("test.db")
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

However, instead of creating Buckets for each one of our models, storm takes care of this for us based on our model declarations. With storm, we also don't have to serialize our models in order to save them.

Again, let's start with Config. Adding a new entry just consists of creating the model object and calling `db.Save` - easy. No fiddling around with serialization or buckets.

```go
func addConfig(db *storm.DB, height float64, birthday time.Time) error {
    config := Config{Height: height, Birthday: birthday}
    err := db.Save(&config)
    if err != nil {
     return fmt.Errorf("could not save config, %v", err)
    }
    return nil
}
```


Querying is also straightforward. We want the latest entry and there are multiple ways to achieve this, one way is to query in reverse order and limit to 1:

```go
    var configs []Config
    err = db.All(&configs, storm.Limit(1), storm.Reverse())
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(configs)
```


Next up is weight, which is not much different. Adding works the same way as with Config:

```go
func addWeight(db *storm.DB, value float64, date time.Time) error {
    weight := Weight{Weight: value, Date: date}
    err := db.Save(&weight)
    if err != nil {
        return fmt.Errorf("could not save weight, %v", err)
    }
    return nil
}
```


In order to query the weight timeline, we just query all weights in reverse order. The current weight would be the same query as for the latest config:

```go
    var weights []Weight
    err = db.All(&weights, storm.Reverse())
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(weights)
```


In the post about BoltDB, entries were more complex, as we had to use the `RFC3339` date format to save them. With storm, we can just create our data model with a normal `time.Time` and everything is fine:

```go
func addEntry(db *storm.DB, food string, calories int, date time.Time) error {
    entry := Entry{Food: food, Calories: calories, Date: date}
    err := db.Save(&entry)
    if err != nil {
        return fmt.Errorf("could not save entry, %v", err)
    }
    return nil
}
```


Fetching entries within a time-span, e.g.: to display all entries of a single day, or within a week was also rather cumbersome. Storm provides several ways to achieve this more using it's simple and advanced query APIs.

For example, fetching today's entries using `db.Range()`:

```go
    var today []Entry
    err = db.Range("Date", time.Now().AddDate(0, 0, -1), time.Now().AddDate(0, 0, 1), &today)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Entries from Today:")
    fmt.Println(today)
```


Or, fetching all entries between 3 and 1 days ago using the `q` query API:

```go
    var twoDaysAgo []Entry
    query := db.Select(q.Gt("Date", time.Now().AddDate(0, 0, -3)), q.Lt("Date", time.Now().AddDate(0, 0, -1)))
    err = query.Find(&twoDaysAgo)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Entries from Two Days Ago:")
    fmt.Println(twoDaysAgo)
```


Pretty neat, and date handling works out of the box. We could also add more clauses using `q.And()`, `q.Not()`, `q.Or()` etc. to make more fine-grained queries.

Using storm, it would also be trivial to update and/or delete data entries using the query API's `db.Update`, `db.UpdateField` and `db.DeleteStruct` methods.

That's it. You can find the full code [here](https://github.com/zupzup/boltdb-storm-example).

## Conclusion 

Even after this pretty simple example I must say I like storms API and how much easier it is to do complex queries than with BoltDB alone.

However, this post barely scratched the surface of storm's advanced [query api](https://godoc.org/github.com/asdine/storm/q) and we didn't use any indexes, so there's quite a bite more to storm than what our little example here showed off.

As I wrote in the beginning - if the problem at hand isn't a good fit for BoltDB, you should probably use another data store, but for cases where a nice query API saves time and the abstraction cost isn't that relevant, storm seems like a solid and well documented solution.

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/boltdb-storm-example)
* [boltdb](https://github.com/boltdb/bolt)
* [storm](https://github.com/asdine/storm)
