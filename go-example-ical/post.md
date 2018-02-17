This post will show a short example of how to create an `.ics` feed from a Go web server. The data source will be a fake REST endpoint created with [mocky](https://www.mocky.io/).

The data will also be cached in memory, so we don't have to call the API all the time and are able to provide fast response times.

To create the feed itself, the [goics](https://github.com/jordic/goics) library will be used, which helps with the encoding.

There will be two endpoints on the server:

* `POST /feedURL` - Creates a new, randomly generated feedURL and initializes the feed for the token
* `GET /feed/{token}` - Returns the feed for the given token and, if the cache expired, re-creates it 

With all that out of the way, let's get to it!

## Implementation

First off, let's take a look at our data source. We will use a fake-JSON response from `mocky` and the JSON looks like this: 

```json
[
    {
        "dateStart": "2018-02-01T13:30:01+00:00",
        "dateEnd": "2018-02-01T12:30:01+00:00",
        "description": "dentist"
    },
    {
        "dateStart": "2018-02-02T13:21:01+00:00",
        "dateEnd": "2018-02-02T15:51:01+00:00",
        "description": "gym"
    },
    {
        "dateStart": "2018-02-09T13:21:01+00:00",
        "dateEnd": "2018-02-09T14:21:01+00:00",
        "description": "meeting"
    },
    {
        "dateStart": "2018-02-09T15:21:01+00:00",
        "dateEnd": "2018-02-09T17:41:01+00:00",
        "description": "cooking class"
    },
    {
        "dateStart": "2018-02-11T13:21:01+00:00",
        "dateEnd": "2018-02-11T18:21:01+00:00",
        "description": "gym"
    },
    {
        "dateStart": "2018-02-12T13:21:01+00:00",
        "dateEnd": "2018-02-12T15:21:01+00:00",
        "description": "shopping"
    },
    {
        "dateStart": "2018-02-14T19:00:01+00:00",
        "dateEnd": "2018-02-14T21:00:01+00:00",
        "description": "valentines"
    }
]
```

To keep things simple, the data is already well formed and our aim will be to create calendar-entries with the `description` as the title.

With this data, we can create our data model. An `Entry` struct for parsing the JSON and a `Feed` struct as a container for caching a created feed should suffice:

```go
// Feed is an iCal feed
type Feed struct {
    Content   string
    ExpiresAt time.Time
}

// Entry is a time entry
type Entry struct {
    DateStart   time.Time `json:"dateStart"`
    DateEnd     time.Time `json:"dateEnd"`
    Description string    `json:"description"`
}

// Entries is a collection of entries
type Entries []*Entry
```

There is also an `Entries` collection-type which we will need for the `.ics` encoding, but we will deal with this later on.

The next step is to create a simple Go webserver with the above mentioned routes:

```go

const feedPrefix = "/feed/"
const expirationTime = 20 * time.Minute // caching time

func main() {
    cache := make(map[string]*Feed)

    mux := http.NewServeMux()
    mux.HandleFunc("/feedURL", feedURL(cache))
    mux.HandleFunc(feedPrefix, feed(cache))

    log.Print("Server started on localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

We create a simple "cache", which in our case is just a `map[string]*Feed` and register the handler functions for the two routes.

Let's take a look at `feedURL` first:

```go
func feedURL(cache map[string]*Feed) http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := randomToken(20)
        _, err := createFeedForToken(token, cache)
        if err != nil {
            writeError(http.StatusInternalServerError, "Could not create feed", w, err)
            return
        }
        writeSuccess(fmt.Sprintf("FeedToken: %s", token), w)
    })
}
```

This handler creates a random token using a simple helper function with `crypto/rand`, calls `createFeedForToken`, which we will look at later, creating a new feed for the given token and returns the token to the user.

The second handler, `/feed/{token}` is bit more in involved:

```go
func feed(cache map[string]*Feed) http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-type", "text/calendar")
        w.Header().Set("charset", "utf-8")
        w.Header().Set("Content-Disposition", "inline")
        w.Header().Set("filename", "calendar.ics")

        var result string
        token := parseToken(r.URL.Path)

        feed, ok := cache[token]
        if !ok || feed == nil {
            writeError(http.StatusNotFound, "No Feed for this Token", w, errors.New("No Feed for this Token"))
            return
        }

        result = feed.Content
        if feed.ExpiresAt.Before(time.Now()) {
            newFeed, err := createFeedForToken(token, cache)
            if err != nil {
                writeError(http.StatusInternalServerError, "Could not create feed", w, err)
                return
            }
            result = newFeed.Content
        }

        writeSuccess(result, w)
    })
}
```

After setting the standard headers for the `.ics` response and parsing the provided token, we check if there is a cache-entry for the given token. If there is not, we return a `404` error.

If there is a feed for the token, but it is expired, we re-create the feed using `createFeedForToken` and return the newly created feed. 

We just return the feed from the cache if there is an entry and it is still valid.

Now, let's take a look at the `createFeedForToken` function:

```go
func createFeedForToken(token string, cache map[string]*Feed) (*Feed, error) {
    res, err := fetchData()
    if err != nil {
        return nil, err 
    }

    b := bytes.Buffer{}
    goics.NewICalEncode(&b).Encode(res)

    feed := &Feed{
        Content: b.String(),
        ExpiresAt: time.Now().Add(expirationTime)
    }
    cache[token] = feed

    return feed, nil
}
```

The `fetchData` function does nothing fancy. It calls the `mocky` URL and unmarshals the resulting JSON to a list of `Entry` structs, handling all possible errors. 

Then, we create the actual `.ics` feed using `goics.NewICalEncode().Encode()`, put it into the cache with the given token and return the feed.

In order to be able to use our list of `Entry` structs as a valid input for `goics.NewICalEncode(&b).Encode()`, the `Entries` type needs to implement the `ICalEmitter` interface:

```go
// EmitICal implements the interface for goics
func (e Entries) EmitICal() goics.Componenter {
    c := goics.NewComponent()
    c.SetType("VCALENDAR")
    c.AddProperty("CALSCAL", "GREGORIAN")

    for _, entry := range e {
        s := goics.NewComponent()
        s.SetType("VEVENT")

        k, v := goics.FormatDateTimeField("DTSTART", entry.DateStart)
        s.AddProperty(k, v)
        k, v = goics.FormatDateTimeField("DTEND", entry.DateEnd)
        s.AddProperty(k, v)

        s.AddProperty("SUMMARY", entry.Description)
        c.AddComponent(s)
    }
    return c
}
```

The method uses `goics` helpers for creating the `.ics` output as shown in the library documentation.

Now, if we call `/feedURL` and use the resulting token as an input to `/feed/{token}`, we should get a valid `calendar.ics` file, which can be imported into apps like iCalendar, Google Calendar etc.

Also notice how the first request to `/feedURL` which initializes the feed takes about ~100 ms and the following requests to `/feed/`, which return the cached result, are very quick.

That's it. You can find the full code [here](https://github.com/zupzup/example-go-ical).

## Conclusion 

This was another short example in Go of a basic feature which is often implemented for web applications. I didn't find many Go libraries for `iCal`, but the one I used worked well and was sufficient for this use-case.

More generally, I believe this example illustrates how powerful the Go standard library is and how simple a feature such as this can be implemented using it.

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/example-go-ical)
* [mocky](https://www.mocky.io/)
* [goics](https://github.com/jordic/goics)
