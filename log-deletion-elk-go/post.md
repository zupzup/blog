The so-called ELK stack, consisting of the tools [Elasticsearch](https://www.elastic.co/jp/products/elasticsearch), [Logstash](https://www.elastic.co/jp/products/logstash) and [Kibana](https://www.elastic.co/jp/products/kibana) is a great way to handle logs from a distributed system.

I won't go into any detail on the ELK stack in this post, but suffice to say that it's quite powerful, relatively simple to set up and that it scales and works nicely even with the free version. The ELK stack provides mechanisms to collect, structure, persist and analyze logs from different sources. However, with a non-trivial amount of services sending logs all the time, it's important to clean up once in a while and remove some old log files so we don't run out of disk space.

There is no "built-in" functionality for this in the ELK stack (at least I couldn't find one) but, depending on how you set up your log-collection with `logstash`, it boils down to deleting indices from `elasticsearch`, which can be done using `cURL`.

So, let's say we have two different sources of logs - from our `gateway` servers  and from our `application` servers. In `logstash`, we could use one of the many `inputs` available such as filebeat and send the logs to `elasticsearch` with a config such as this:

```bash
output {
  elasticsearch {
    hosts => [ "elasticsearch:9200" ]
    index => "logstash-application-%{+YYYY.MM.dd}"
  }
}
```

This means, that the incoming logs will be sent to `elasticsearch` in the format specified after `index`. It's a good idea (at least at our scale - this might differ for larger systems), to simply create an index for every day and persist the logs in there. This could, of course, also be done hourly or monthly or in any other arbitrary way. What is important though, in order to be able to delete logs by how old they are, is to include some kind of timestamp in.

In the following two sections, we will conceptualize a small script, which will make it easy for us to delete old logs from different sources with a way to specify how far back we want to go.

Let's get started.

## Concept 

So, let's call our script `logdeleter`. Basically, we want to specify some concept of time - a maximum age of logs we want to keep (e.g. 1 month). There are several ways to do this, but I think 2 parameters - one for the value and one for the unit will suffice here:

* `--v` - the value of time
* `--u` - the unit of time

Also, we want to be able to specify different kinds of logs, so in our example, we will add a prefix parameter, which will delete only the logs with an index starting with the given prefixes:

* `--p` - the prefixes of the indices to delete (everything before the date part)

In a more complex version, this could also be a RegEx. For testing purposes, when I write deletion scripts or destructive operations in general, I like to add a dry-run option, which shows the user what would happen, if the script were run with the given parameters:

* `--dr` - dryRun mode - doesn't actually delete indices, but shows the indices which would have been deleted

Alright - with these parameters, we should be able to do the following and much more:

```bash
// 1 month
./logdeleter --v 1 --u m --p logstash-application- --p logstash-gateway-

// 1 week
./logdeleter --v 1 --u w --p logstash-application- --p logstash-gateway-

// 20 days
./logdeleter --v 20 --u d --p logstash-application- --p logstash-gateway-

// 1 year
./logdeleter --v 1 --u y --p logstash-application- --p logstash-gateway-

// 1 year dry-run
./logdeleter --dr --v 1 --u y --p logstash-application- --p logstash-gateway-
```

This should give us more than enough flexibility for most use-cases.

Let's start coding this up using Go.

## Code

Ok, first things first - dependencies. There isn't much we need and we could easily write this just using the standard library, but the [go elasticsearch client](https://github.com/olivere/elastic) has worked well for me in the past and I always want nice logs, so `logrus` will be in there as well.

The most "complex" part of this simple script is the parameter parsing:

```go
type arrayFlags []string

func (i *arrayFlags) String() string {
    return strings.Join(*i, ",")
}

func (i *arrayFlags) Set(value string) error {
    *i = append(*i, value)
    return nil
}

var dryRun bool
var value int
var unit string
var prefixes arrayFlags

func init() {
    flag.IntVar(&value, "v", 1, "Delete logs older than this value together with the unit, e.g. 1")
    flag.StringVar(&unit, "u", "m", "Delete logs older than this unit together with the value, e.g. m for month")
    flag.Var(&prefixes, "p", "Prefixes (part before the date) of the indices, which should be deleted, e.g. logstash-application-")
    flag.BoolVar(&dryRun, "dr", false, "Run the script without actually deleting anything")
}

func main() {
    flag.Parse()

    if value <= 0 {
        log.Fatal("You need to specify a valid time after which logs are deleted, e.g. --v=1 --u=w for 1 week\n")
    }

    if unit != "m" && unit != "w" && unit != "d" && unit != "y" {
        log.Fatal("You need to specify a valid unit for the time after which logs are deleted, e.g. --v=1 --u=w for 1 week. Valid units are d, w, m, y\n")
    }

    if len(prefixes) == 0 {
        log.Fatal("You need to specify prefixes for which logs should be deleted, e.g. --p=logstash-application --p=logstash-gateway\n")
    }
    ...
}
```

Alright. So this is mostly pretty basic usage of the `flag` package - we define our parameters in the `init` function and then call `flag.Parse()` in order to set them to the globals we defined above.

However, in this example, the `prefix / --p` parameter is actually a bit more complex as it should be possible to add multiple values for this flag. This is where the `arrayFlags` type comes from, this is a custom flag type we create, which handles reading and setting multiple values for a parameter.

After parsing the flags, we add some checks and inform the user, if the given parameters don't work. For example an invalid time unit or value, or if there are simply no prefixes.

The next goal is to get a connection to `elasticsearch` and to query all existing index-names, so we can find the ones we want to delete.

```go
ESHost := os.Getenv("ES_HOST")
if ESHost == "" {
    ESHost = "http://127.0.0.1:9200"
}

ctx := context.Background()
client, err := elastic.NewClient(elastic.SetURL(ESHost), elastic.SetSniff(false))
if err != nil {
    log.Fatalf("Could not connect to ElasticSearch: %v\n", err)
}

log.Infof("LogDeleter started, deleting logs older than %d%s with prefixes %s", value, unit, prefixes)

names, err := client.IndexNames()
if err != nil {
    log.Fatalf("Could not fetch indices from ELasticSearch: %v\n", err)
}
```

Nothing special happening here - we just use the API of the `elasticsearch` client and that's that.

Now comes the "central piece" of the script - we iterate all the indices, filter by our given `prefixes`, parse the filtered indices' date and check, if the given date is after the cut-off date calculated using the `time` parameters. Also, we check if we're on a dry-run and if so, we don't actually delete anything, but print the indices, which would have been deleted.

```go
for _, index := range names {
    if hasCorrectPrefix(index, prefixes) {
        indexDate := trimPrefix(index, prefixes)
        date, err := time.Parse("2006.01.02", indexDate)
        if err != nil {
            log.Errorf("Index %s's date could not be parsed", index)
        }
        if shouldBeDeleted(date, value, unit) {
            if !dryRun {
                _, err := client.DeleteIndex(index).Do(ctx)
                if err != nil {
                    log.Errorf("Could not delete index %s, %v\n", index, err)
                } else {
                    log.Infof("Deleted Index: %s\n", index)
                }
            } else {
                log.Infof("DryRun - would have deleted Index: %s\n", index)
            }
        }
    }
}
```

There are several helper-functions in the above snippet. The first one is `hasCorrectPrefix`, which simply checks if the given index is prefixed with any of the given `prefixes`:

```go
func hasCorrectPrefix(index string, prefixes []string) bool {
    result := false
    for _, prefix := range prefixes {
        if strings.HasPrefix(index, prefix) {
            return true
        }
    }
    return result
}
```

Then, that `prefix` is removed, in order to get the `date` part of the index name using `trimPrefix`:

```go
func trimPrefix(index string, prefixes []string) string {
    for _, prefix := range prefixes {
        if strings.HasPrefix(index, prefix) {
            return strings.TrimPrefix(index, prefix)
        }
    }
    return index
}
```

These two functions could have been done in one loop, but as performance really isn't the most important goal here, I split it up for readability.

After parsing the date, we use the `shouldBeDeleted` helper function, in order to find out, if the current index should be removed:

```go
func shouldBeDeleted(date time.Time, value int, unit string) bool {
    if calculateTargetDate(date, value, unit).After(time.Now()) {
        return false
    }
    return true
}

func calculateTargetDate(date time.Time, value int, unit string) time.Time {
    if unit == "d" {
        return date.AddDate(0, 0, value)
    }
    if unit == "w" {
        return date.AddDate(0, 0, value*7)
    }
    if unit == "m" {
        return date.AddDate(0, value, 0)
    }
    if unit == "y" {
        return date.AddDate(value, 0, 0)
    }
    return date
}
```

So, first off the date of the index is used to calculate a `target date` - which is the date, given the time parameters, where this log would still be ok and wouldn't have to to be deleted. In order to do that, we parse the unit parameter and add days/weeks/years according to it to the index date.
Then, if the index date is `After` the current time, we know the index is still ok and we do nothing, otherwise, we delete it.

The actual deletion is pretty simple, we just call

```go
_, err := client.DeleteIndex(index).Do(ctx)
```

and handle the error.

That's it. :)

The full code can be found [here](https://gist.github.com/zupzup/775ce8f5d364ec9ea5f0862ca7c4d0de)

## Conclusion

The ELK stack has been serving us really well in terms of setting up a pipeline for analyzing and storing the logs of our microservice system. The motivation to write this short script was mostly to have some fun in Go, but also because it's nice to have maintainable ops tools instead of bash scripts.

Go is a great language for small ops programs such as this due to cross-compilation and the possibility to create static binaries which can be distributed without any trouble.

#### Resources

* [Full Code](https://gist.github.com/zupzup/775ce8f5d364ec9ea5f0862ca7c4d0de)
* [go elasticsearch client](https://github.com/olivere/elastic)
* [Elasticsearch](https://www.elastic.co/jp/products/elasticsearch)
* [Logstash](https://www.elastic.co/jp/products/logstash)
* [Kibana](https://www.elastic.co/jp/products/kibana)
