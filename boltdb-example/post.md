Recently, while developing a command line application with Go, which I planned to release for multiple platforms (Windows, OS X, Linux), I stumbled upon a bit of a problem because I chose SQLite for storing data.

Unfortunately, the [go sqlite3 package](https://github.com/mattn/go-sqlite3) is a CGO package, which makes it a pain for cross-compilation, especially in a simple and automated way.

For this reason, I looked for alternatives and finally came around to try out [bolt](https://github.com/boltdb/bolt), a simple key/value store written in pure Go. My Data Model isn't very sophisticated, my main reason for using SQLite was because I wanted to use a single file for storage.

In this post, we'll look at one way to implement a given Data Model using bolt. The use-case is that of a (possibly offline), standalone application for a single user.

Let's start with the Data Model we want to use.

## Data Model

```go
type Config struct {
    Birthday time.Time
    Height float
}
```

```go
type Weight struct {
    Date time.Time
    Weight float64
}
```

```go
type Entry struct {
    Date time.Time
    Food string
    Calories int
}
```

This simplified Data Model could be used for a calories-tracker application or something similar. The specific use-case isn't all that relevant here. What's relevant are the different ways in which we want to store and query data.

Basically, we want to store the current configuration, the weight changes over time and food/calories entries. With this data, it would be possible to calculate an approximate Calories Budget or to display a timeline of the user's eating behaviour. 

Next, let's look at one way to realize this using bolt.

## Implementation

First, we set up the database:

```go
func setupDB() (*bolt.DB, error) {
    db, err := bolt.Open("test.db", 0600, nil)
    if err != nil {
        return nil, fmt.Errorf("could not open db, %v", err)
    }
    err = db.Update(func(tx *bolt.Tx) error {
        root, err := tx.CreateBucketIfNotExists([]byte("DB"))
        if err != nil {
        return fmt.Errorf("could not create root bucket: %v", err)
        }
        _, err = root.CreateBucketIfNotExists([]byte("WEIGHT"))
        if err != nil {
        return fmt.Errorf("could not create weight bucket: %v", err)
        }
        _, err = root.CreateBucketIfNotExists([]byte("ENTRIES"))
        if err != nil {
        return fmt.Errorf("could not create days bucket: %v", err)
        }
        return nil
    })
    if err != nil {
        return nil, fmt.Errorf("could not set up buckets, %v", err)
    }
    fmt.Println("DB Setup Done")
    return db, nil
}
```

In BoltDB, keys and values are simple `[]byte` arrays. Nested structures can be achieved using `Buckets`. All keys within a `Bucket` must be unique. We create Buckets for all our models except `Config`, which we will just put into the root Bucket.

Let's start with the Config Model, which is simple - we just store it once and update it, if it changes:

```go
func setConfig(db *bolt.DB, config Config) error {
    confBytes, err := json.Marshal(config)
    if err != nil {
        return fmt.Errorf("could not marshal config json: %v", err)
    }
    err = db.Update(func(tx *bolt.Tx) error {
        err = tx.Bucket([]byte("DB")).Put([]byte("CONFIG"), confBytes)
        if err != nil {
            return fmt.Errorf("could not set config: %v", err)
        }
        return nil
    })
    fmt.Println("Set Config")
    return err
}

```

And we can retrieve it like this:

```go
err = db.View(func(tx *bolt.Tx) error {
    conf := tx.Bucket([]byte("DB")).Get([]byte("CONFIG"))
    fmt.Printf("Config: %s\n", conf)
    return nil
})
```

Weight is also simple - we use the date of the weigh-in as a key and just store the new weight at this time. The last entry is always the current weight and otherwise we fetch the whole timeline.

```go
func addWeight(db *bolt.DB, weight string, date time.Time) error {
    err := db.Update(func(tx *bolt.Tx) error {
        err := tx.Bucket([]byte("DB")).Bucket([]byte("WEIGHT")).Put([]byte(date.Format(time.RFC3339)), []byte(weight))
        if err != nil {
            return fmt.Errorf("could not insert weight: %v", err)
        }
        return nil
    })
    fmt.Println("Added Weight")
    return err
}
```

Retrieving all Weights:

```go
err = db.View(func(tx *bolt.Tx) error {
    b := tx.Bucket([]byte("DB")).Bucket([]byte("WEIGHT"))
    b.ForEach(func(k, v []byte) error {
        fmt.Println(string(k), string(v))
        return nil
    })
    return nil
})
```

Entries are a little more complex, because we want to be able to fetch entries grouped by the days they were booked on. It would also be interesting to be able to fetch all days with their entries for a week, month or some other defined timespan.

For this reason, we use the `RFC3339` formatted timestamp as a key. This way, we can compare it's byte value and query by date.

```go
func addEntry(db *bolt.DB, calories int, food string, date time.Time) error {
    entry := Entry{Calories: calories, Food: food}
    entryBytes, err := json.Marshal(entry)
    if err != nil {
       return fmt.Errorf("could not marshal entry json: %v", err)
    }
    err = db.Update(func(tx *bolt.Tx) error {
        err := tx.Bucket([]byte("DB")).Bucket([]byte("ENTRIES")).Put([]byte(date.Format(time.RFC3339)), entryBytes)
        if err != nil {
            return fmt.Errorf("could not insert entry: %v", err)
        }
        return nil
    })
    fmt.Println("Added Entry")
    return err
}

```

Fetching all entries within a time-span, e.g.: last seven days:

```go
err = db.View(func(tx *bolt.Tx) error {
    c := tx.Bucket([]byte("DB")).Bucket([]byte("ENTRIES")).Cursor()
    min := []byte(time.Now().AddDate(0, 0, -7).Format(time.RFC3339))
    max := []byte(time.Now().AddDate(0, 0, 0).Format(time.RFC3339))
    for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
        fmt.Println(string(k), string(v))
    }
    return nil
})
```

That's it. You can find the full code [here](https://github.com/zupzup/boltdb-example).

## Conclusion 

I really enjoyed using Bolt. Although I only used it for a pretty narrow use-case, it's simplicity was quite refreshing. For me, the fact that it's pure Go is also a nice benefit, but this probably won't be as important for many other applications.

In a future blog post I want to take a look at [storm](https://github.com/asdine/storm), a Toolkit/ORM for Boltdb and maybe try to take on some more complex use-cases.

In any case, Bolt is simple to use, well documented and definitely worth a try. :)

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/boltdb-example)
* [boltdb](https://github.com/boltdb/bolt)
