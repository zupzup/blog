This post describes how to create a small Go HTTP server, which is able to expose data from [Elasticsearch](https://www.elastic.co) on a [Prometheus](https://prometheus.io/) `/metrics` endpoint.

This can be useful if, for example, you collect logs of a web application using the [ELK](https://www.elastic.co/webinars/introduction-elk-stack) stack, in which case the logs will be saved in Elasticsearch.

A sample use-case would be to analyze the collected logs in regards to returned response codes or response time of single requests.

In this post, I assume the reader to have at least basic knowledge of Elasticsearch as well as Prometheus in regards to what they are and what these tools are used for, as I won't go into any detail on these topics.

TBD example description

Let's get started!

## Implementation 

TBD

```go
type GatewayLog struct {
    Timestamp              string  `json:"@timestamp"`
    Env                    string  `json:"env"`
    StatusCode             int     `json:"backend_status_code"`
    BackendProcessingTime  float64 `json:"backend_processing_time"`
}

func main() {
    ESHost := "http://127.0.0.1:9200"
    GatewayLogIndex := "some_logging_index" 
    LogDelay := 5 * time.Minute
    UpdateIntervalEnv := 30 * time.Second 
```

```go
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

```go
    // fetch data
    fetchDataFromElasticSearch(
        ctx,
        UpdateInterval,
        LogDelay,
        GatewayLogIndex,
        client,
        statusCodeCollector,
        responseTimeCollector,
    )

    r := chi.NewRouter()
    r.Use(render.SetContentType(render.ContentTypeJSON))
    r.Handle("/metrics", promhttp.Handler())
    log.Infof("ElasticSearch-Exporter started on localhost:8092")
    log.Fatal(http.ListenAndServe(":8092", r))
}
```

```go
func fetchDataFromElasticSearch(
    ctx context.Context,
    UpdateInterval time.Duration,
    LogDelay time.Duration,
    GatewayLogIndex string,
    client *elastic.Client,
    statusCodeCollector *prometheus.CounterVec,
    responseTimeCollector *prometheus.SummaryVec,
) {
    ticker := time.NewTicker(UpdateInterval)
    go func() {
        for range ticker.C {
            now := time.Now()
            lastUpdate := now.Add(-LogDelay - UpdateInterval)

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

```go
func handleLogResult(l GatewayLog, statusCodeCollector *prometheus.CounterVec, responseTimeCollector *prometheus.SummaryVec) {
    trackStatusCodes(statusCodeCollector, l.StatusCode, l.Env)
    responseTimeCollector.WithLabelValues(l.Env).Observe(l.BackendProcessingTime)
}
```

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

That's it. Here is a link to the [Full Code](https://gist.github.com/zupzup/5771edb745b0fbafb937a0bd4c37fd3b)

## Conclusion

TBD

#### Resources

* [Code on GitHub](https://gist.github.com/zupzup/5771edb745b0fbafb937a0bd4c37fd3b)
* [prometheus](https://prometheus.io/)
* [elasticsearch](https://www.elastic.co)
* [go elasticsearch client](https://github.com/olivere/elastic)
* [go prometheus client](https://github.com/prometheus/client_golang)
* [chi router](https://github.com/go-chi/chi)
