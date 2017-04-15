Recently, while developing a command line application with Go, which I planned to release for multiple platforms (Windows, OS X, Linux), I stumbled upon a bit of a problem because I chose SQLite for storing data.

Unfortunately, the [go sqlite3 package](https://github.com/mattn/go-sqlite3) is a CGO package, which makes it a pain for cross-compilation, especially in a simple and automated way.

For this reason, I looked for alternatives and finally came around to try out [bolt](https://github.com/boltdb/bolt), a simple key/value store written in pure Go. My Data Model isn't very sophisticated, my main reason for using SQLite was because I wanted to use a single file for storage.

In this post, we'll look at one way to implement a given Data Model using bolt and also how the [storm](https://github.com/asdine/storm) toolkit for bolt can be helpful for more advanced problems.

Let's start with the Data Model we want to use.

## Data Model

```go
type Config struct {
    Birthday time.Time
    Height float
    Gender string
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

Basically, we want to store the current configuration, the weight changes over time and food/calories entries. With this data it would be possible to calculate an approximate Calories Budget or to display a timeline of the user's eating behaviour. 

Config is simple - we just store it once and update it, if it changes

Weight is also pretty simple - we use the date of the weigh-in as a key and just store the new weight at this time. The last entry is always the current weight and otherwise we fetch the whole timeline.

Entries are a little more complicated, because we want to be able to fetch entries grouped by the days they were booked on, as we're interested in the daily calories budget. it would also be interesting to be able to fetch all days with their entries for a week or month or a defined timespan.

Next, let's look at one way to realize this using bolt.

## Implementation

TBD boltdb example

## Storm 

TBD storm

## Conclusion 

TBD

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/boltdb-example)
* [boltdb](https://github.com/boltdb/bolt)
* [storm](https://github.com/asdine/storm)
