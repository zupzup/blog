This post describes how to create a small Go HTTP server, which is able to expose data from [Elasticsearch](https://www.elastic.co) on a [Prometheus](https://prometheus.io/) `/metrics` endpoint.

This can be useful if, for example, you collect logs of a web application using the [ELK](https://www.elastic.co/webinars/introduction-elk-stack) stack, in which case the logs will be saved in Elasticsearch.

A sample use-case would be to analyze the collected logs in regards to returned response codes or response time of single requests.

In this post, I assume the reader to have at least basic knowledge of Elasticsearch as well as Prometheus in regards to what they are and what these tools are used for, as I won't go into any detail on these topics.

In the following example code, we will take a look at how to interact with an Elasticsearch cluster using [elastic](https://github.com/olivere/elastic), as well as how to expose metrics for prometheus using the official [prometheus go client](https://github.com/prometheus/client_golang).

The idea behind the example is that we have an index `some_logging_index` with structured logging data on Elasticsearch, including the server `environment` the log is coming from, the `processing_time` of requests as well as their `status_code`.

Our goal is to make these data-points available to prometheus, so we can analyze the data and/or create alerts based on it. (e.g. if the 99th percentile of response-time is above a certain threshold)

This example isn't necessarily there to copy and run with your own data, as that will require a bit of setup and knowledge of ES and prometheus, but rather to see how one could go about doing something like this using Go.

Let's get started!

## Implementation 

First off, we define the data structure for our structured log as described above:

```go
type GatewayLog struct {
    Timestamp              string  `json:"@timestamp"`
    Env                    string  `json:"env"`
    StatusCode             int     `json:"backend_status_code"`
    BackendProcessingTime  float64 `json:"backend_processing_time"`
}
```

The next step is to define some initial values, such as the update interval for fetching data and the Elasticsearch host as well as to set up the connection to Elasticsearch using `elastic`:

```go
func main() {
    ESHost := "http://127.0.0.1:9200"
    GatewayLogIndex := "some_logging_index" 
    UpdateIntervalEnv := 30 * time.Second 
    ctx := context.Background()
    log.Info("Connecting to ElasticSearch..")
    var client *elastic.Client
    for {
        esClient, err := elastic.NewClient(elastic.SetURL(ESHost), elastic.SetSniff(false))
        if err != nil {
            log.Errorf("Could not connect to ElasticSearch: %v\n", err)
            time.Sleep(1 * time.Second)
            continue
        }
        client = esClient
        break
    }

    info, _, err := client.Ping(ESHost).Do(ctx)
    if err != nil {
        log.Fatalf("Could not ping ElasticSearch %v", err)
    }
    log.Infof("Connected to ElasticSearch with version %s\n", info.Version.Number)
```

There isn't much happening here - if we can't connect, we retry, otherwise we continue.

After having our Elasticsearch setup ready, we also need to initialize our prometheus metrics:

```go
    statusCodeCollector := prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "gateway_status_code",
        Help: "Status Code of Gateway",
    }, []string{"env", "statuscode", "type"})

    responseTimeCollector := prometheus.NewSummaryVec(prometheus.SummaryOpts{
        Name: "gateway_response_time",
        Help: "Response Time of Gateway",
    }, []string{"env"})

    if err := prometheus.Register(statusCodeCollector); err != nil {
        log.Fatal(err, "could not register status code 500 collector")
    }
    if err := prometheus.Register(responseTimeCollector); err != nil {
        log.Fatal(err, "could not register response time collector")
    }
```

We use two different kinds of metrics here, `Counter` and `Summary`. A counter is, not surprisingly, just a simple numerical metric, which counts occurrences up. This is perfect for the `status_code` of requests, as it gives us the overall distribution, as well as the possibility to query the difference in the counter over a timespan from prometheus. We categorize the `status_code` by the `environment` it is coming from as well as the `type` (e.g. 2xx, 5xx...).

The second metric is a `Summary` and we will use it for the `response_time`. A summary is a time-series based approach, which automatically puts the values into quantile-buckets (default: 0,5 0,9 and 0,99). This is exactly what we want, as we're interested in querying whether for example, the 95th percentile of response times is below a certain threshold.

After defining the prometheus metrics, we register them, register and endpoint for the metrics at `/metrics` with our router and continue.

```go
    r := chi.NewRouter()
    r.Use(render.SetContentType(render.ContentTypeJSON))
    r.Handle("/metrics", promhttp.Handler())
    log.Infof("ElasticSearch-Exporter started on localhost:8092")
    log.Fatal(http.ListenAndServe(":8092", r))
}
```

The next and most complex step in this small program is to actually fetch data from Elasticsearch and add it to our metrics.

For this purpose, we will use the following function:

```go
func fetchDataFromElasticSearch(
    ctx context.Context,
    UpdateInterval time.Duration,
    GatewayLogIndex string,
    client *elastic.Client,
    statusCodeCollector *prometheus.CounterVec,
    responseTimeCollector *prometheus.SummaryVec,
) {
    ticker := time.NewTicker(UpdateInterval)
    go func() {
        for range ticker.C {
            now := time.Now()
            lastUpdate := now.Add(-UpdateInterval)

            rangeQuery := elastic.NewRangeQuery("@timestamp").
                Gte(lastUpdate).
                Lte(now)

            log.Info("Fetching from ElasticSearch...")
            scroll := client.Scroll(GatewayLogIndex).
                Query(rangeQuery).
                Size(5000)

            scrollIdx := 0
            for {
                res, err := scroll.Do(ctx)
                if err == io.EOF {
                    break
                }
                if err != nil {
                    log.Errorf("Error while fetching from ElasticSearch: %v", err)
                    break
                }
                scrollIdx++
                log.Infof("Query Executed, Hits: %d TookInMillis: %d ScrollIdx: %d", res.TotalHits(), res.TookInMillis, scrollIdx)
                var typ GatewayLog
                for _, item := range res.Each(reflect.TypeOf(typ)) {
                    if l, ok := item.(GatewayLog); ok {
                        handleLogResult(l, statusCodeCollector, responseTimeCollector)
                    }
                }
            }
        }
    }()
}
```

Alright, so it wasn't that complex after all. We create a timer, which ticks in a given interval using `time.NewTicker`. Then we iterate over this ticker with Go's pretty `range`-syntax.

For every tick, we calculate when we last updated our data and create an Elasticsearch query, which gives us the logging data from that point in time until now. Keep in mind here, that it is possible that we drop some logs in between the time it takes to execute this, even though it's just a few ms. This is a small trade-off with this implementation, which makes it simpler than trying to keep track of every single log entry. 

We're doing analysis on huge amounts of data here anyway, so dropping a couple of logs here, or there likely won't make that much difference, but it's still good to keep in mind.

Because this query can return quite a lot of data in a production environment, we have to `scroll` through the data. In this case, we scroll by 5000 entries every time and log the progress for every batch of data.

Now comes the interesting part. We iterate through the data, but only use results of type `GatewayLog` - so data which has the above defined fields and call `handleLogResult` for each log:


```go
func handleLogResult(l GatewayLog, statusCodeCollector *prometheus.CounterVec, responseTimeCollector *prometheus.SummaryVec) {
    responseTimeCollector.WithLabelValues(l.Env).Observe(l.BackendProcessingTime)
    trackStatusCodes(statusCodeCollector, l.StatusCode, l.Env)
}
```

That's all there is to it. For the `response_time`, we call `Observe` with the given `BackendProcessingTime`, which puts the data into our summary. For the `status_codes` we have to do a bit more:

```go
func trackStatusCodes(statusCodeCollector *prometheus.CounterVec, statusCode int, env string) {
    if statusCode >= 500 && statusCode <= 599 {
        statusCodeCollector.WithLabelValues(env, strconv.Itoa(statusCode), "500").Inc()
    } else if statusCode >= 200 && statusCode <= 299 {
        statusCodeCollector.WithLabelValues(env, strconv.Itoa(statusCode), "200").Inc()
    } else if statusCode >= 300 && statusCode <= 399 {
        statusCodeCollector.WithLabelValues(env, strconv.Itoa(statusCode), "300").Inc()
    } else if statusCode >= 400 && statusCode <= 499 {
        statusCodeCollector.WithLabelValues(env, strconv.Itoa(statusCode), "400").Inc()
    }
}
```

Here, we categorize the `status_code` in the HTTP statusCode categories (5xx, 4xx...) and call `.Inc()` for every log, increasing the counter. We also label the entries with the `environment`, the actual status code and the `type` of the status code, which enables us to query for 5xx errors from a specific server for example.

That's it. Here is a link to the [Full Code](https://gist.github.com/zupzup/5771edb745b0fbafb937a0bd4c37fd3b)

## Conclusion

It's no wonder Go has so many fans among the Ops-crowd. With seamless cross-compilation and Go's simplicity, it's just a delight to create small tools such as this to streamline and improve your monitoring and operations toolchain.

The used libraries, `elastic` and the prometheus client both have good APIs and fantastic documentation - I had absolutely no issues.

I hope this post was useful, even if it's not really a runnable example per se and needed some previous knowledge, but having built something similar to this recently, I thought it'd be a good idea to share it. :)

#### Resources

* [Code on GitHub](https://gist.github.com/zupzup/5771edb745b0fbafb937a0bd4c37fd3b)
* [prometheus](https://prometheus.io/)
* [elasticsearch](https://www.elastic.co)
* [go elasticsearch client](https://github.com/olivere/elastic)
* [go prometheus client](https://github.com/prometheus/client_golang)
* [chi router](https://github.com/go-chi/chi)
