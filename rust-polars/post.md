*This post was originally posted on the [LogRocket](https://blog.logrocket.com/using-polars-rust-high-performance-data-analysis/) blog on 21.10.2024 and was cross-posted here by the author.*

In this article, we’ll take a look at how to use Rust and the popular [Polars](https://pola.rs/) high-performance DataFrame library to build a basic data analysis application, which exposes data sets and querying capabilities via a REST-based Web API.

We’re also going to build a simple backend web application, which exposes a REST API, which in turn exposes Polars’ powerful and high-performance querying capabilities on a huge dataset, to see how one could go about exploring some interesting data sets.

We’ll dive into Polars concepts later, but for now, let’s jump into data and setup.

## Data

Since we want to do data analysis in this tutorial, we’ll need some data. It would also be great to have at least two different data sources so we can play around with Polar’s data transformation APIs.

For this purpose, we’ll use the MIT licensed [Deutsche Bahn (DB) delays data set](https://www.kaggle.com/datasets/nokkyu/deutsche-bahn-db-delays) from Kaggle, which is a collection of train ride delays in Germany, in CSV format. This data set contains data about train stations, where they are, where the train went before, the amount of delay at arrival and departure and such.

The second data set we’ll use is an Open Database licensed [mapping from zip code to population and area](https://downloads.suche-postleitzahl.org/v2/public/plz_einwohner.csv), based on OpenStreetMap data from [https://www.suche-postleitzahl.org](https://www.suche-postleitzahl.org/), also in CSV format. This data set is basically a mapping of German zip codes to the population and area of these places.

With these two data sets, we can do some analysis on train delays and also attempt to correlate them with a zip code’s population and area to see if, e.g., trains are delayed more in densely populated areas.

## Setup

To follow along, all you need is a reasonably recent Rust installation with 1.81 being the latest one at the time of this writing.

First, create a new Rust project:

```bash
    cargo new rust-polars-example
    cd rust-polars-example
```

Next, edit the Cargo.toml file and add the dependencies you'll need:

```toml
    [dependencies]
    axum = "0.7.5"
    polars = { version = "0.42.0", features = ["lazy", "csv", "json", "parquet", "strings", "regex", "cov", "serde"] }
    serde = { version = "1.0.204", features = ["derive"] }
    serde_json = "1.0.120"
    tokio = { version = "1.39.2", features = ["full"] }
    tracing = "0.1.40"
    tracing-subscriber = "0.3.18"
```

We’ll use [Axum](https://github.com/tokio-rs/axum) with [Tokio](https://tokio.rs/) to build a web backend, [Tracing](https://github.com/tokio-rs/tracing) for logging, and [Serde](https://serde.rs/) for serialization and deserialization.

And of course, we’ll also need Polars. As you can see, there are a lot of features we activated for handling csv, JSON, strings, etc., as well as for IO and correlations. You can find a list of feature flags and what they do in the [documentation](https://docs.rs/crate/polars/latest/features).

Generally, a thing I found when working with Polars was that sometimes, if something doesn’t work, checking if it might be hidden behind a feature flag often led me to the solution.

With the project setup out of the way, let’s get some data into our application and start building a web server for our REST API.

## Loading data

Since we’re building an example application for data analysis, the most important part is getting the data into our application. Fortunately, Polars provides us with plenty of options to load different file formats in different ways, which can also be configurable. So loading our two CSV files is rather easy:

```rust
        // in async fn main() {
        ...
        let now = Instant::now();
        info!("reading delays csv dataframe");

        let df_delays = CsvReadOptions::default()
            .with_has_header(true)
            .try_into_reader_with_file_path(Some("data.csv".into()))
            .expect("can read delays csv")
            .finish()
            .expect("can create delays dataframe");

        let df_cities = CsvReadOptions::default()
            .with_has_header(true)
            .try_into_reader_with_file_path(Some("zip_pop.csv".into()))
            .expect("can read cities csv")
            .finish()
            .expect("can create cities dataframe");

        info!("done reading csv {:?}", now.elapsed());
        ...
```

We use Polars CSV reader, configuring that we indeed have a header and specifying our file name and loading the data in this way.

The `data.csv` file contains the train delays and `zip_pop.csv` contains the mapping from zip codes to population and area data. And while `zip_pop.csv` is only 473 KB big, the train delay data set clocks in at ~770MB, which means loading it takes a few seconds (~10 on my machine).

One way to get around this issue during development is to just use a subset of the data, such as the first 100,000 lines — something that’ll load instantly — and load that instead while building out the queries.

With the data in the system, let’s build our web server and see how we can make the data available to our REST API.

## Web server setup

Since we’re going to expose our data analysis mechanisms via a REST API, we first need to build a simple web server. In this case, we’ll use Axum and Tokio to do so:

```rust
    #[tokio::main]
    async fn main() {
        tracing_subscriber::fmt::init();
        ...
    }
```

We start by initializing the `tracing_subscriber` to set up our logging infrastructure in an async main function.

Then, we define an `AppState` struct, which we’ll use to share the data frames we got from importing the CSV files above with our REST handlers:

```rust
    struct AppState {
        df_delays: DataFrame,
        df_cities: DataFrame,
    }

    impl AppState {
        fn new(df_delays: DataFrame, df_cities: DataFrame) -> Self {
            Self {
                df_delays,
                df_cities,
            }
        }
    }
```

The application state simply holds the two data frames. It is kept inside of an atomic reference counted smart pointer ([std::sync::Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html)), so we can share it safely across threads:


```rust
        // in main
        let app_state = Arc::new(AppState::new(df_delays, df_cities));

        let app = Router::new()
          .route("/delays", get(delays))
          .route("/cities/:zip/population", get(cities_pop))
          .route("/cities/:zip/area", get(cities_area))
          .route("/delays/:zip/", get(delays_by_zip))
          .route("/delays/corr/:field1/:field2/", get(delays_corr))
          .with_state(app_state);
```

We create a new instance of `AppState` wrapped inside an `Arc` and create our Axum router. We define several routes:


- `GET /delays?sort=[Asc/Desc]&limit=[u32]` - an endpoint for fetching delays with optional search and limitation parameters
- `GET /cities/[zip]/population` and `GET /cities/[zip]/area` - endpoints for fetching a zip code’s population and area
- `GET /delays/[zip]/` - an endpoint for fetching delays for a specified zip code
- `GET /delays/corr/[field1]/[field2]/` - an endpoint for calculating the [Pearson Correlation Coefficient](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient) between two fields. Or, in other words, an endpoint to find out if there is a linear relation between the values in two fields (e.g. if there are more delays in areas with a higher population)

We’ll look at the implementation of the handlers within `get(..)` after setting up the rest of the web server.

Next, we create a `TcpListener` listening on `localhost:8000` and serve the app defined with the above router:

```rust
        // in main
        let listener = TcpListener::bind("0.0.0.0:8000")
            .await
            .expect("can start web server on port 8000");

        info!("listening on {:?}", listener.local_addr());

        axum::serve(listener, app)
            .await
            .expect("can start web server");
    }
```

That’s it for the web server setup. Next, let’s implement the first handler `delays`, which implements querying our data set for delays with a certain sorting and limit.

## A simple first query

The first endpoint returns a limited list of delays, sorted either ascending or descending, depending on what the caller passes as query parameters to the endpoint.

To implement this, the first thing we need to do is create a struct for these options:

```rust
    #[derive(Deserialize, Eq, PartialEq)]
    enum SortingOptions {
        Desc,
        Asc,
    }

    #[derive(Deserialize)]
    struct DelayOptions {
        sorting: Option<SortingOptions>,
        limit: Option<u32>,
    }
```

Now in the `delays` handler function, we can use these `DelayOptions` as `Query` parameters:

```rust
    async fn delays(
        query: Query<DelayOptions>,
        app_state: State<Arc<AppState>>,
    ) -> Result<Json<Value>, StatusCode> {
      ...
    }
```

Besides the query parameters, the handler gets passed to our app state, so we can access the loaded data frames.

First, we figure out which sorting and limit options to take for our query:

```rust
        let mut sorting_mode = SortMultipleOptions::default()
            .with_order_descending(true)
            .with_nulls_last(true);
        if let Some(query_sorting) = &query.sorting {
            if *query_sorting == SortingOptions::Asc {
                sorting_mode = SortMultipleOptions::default()
                    .with_order_descending(false)
                    .with_nulls_last(true);
            }
        }
        let mut limit = 10;
        if let Some(query_limit) = query.limit {
            limit = query_limit
        }
```

We default to `Desc` and `limit 10` and otherwise take the supplied values — if the user provides incompatible values, they’ll get an HTTP 400 error.

With the amount of items to return and the sorting order, we can finally get to querying data using Polars:

```rust
        let delays = app_state
            .df_delays
            .clone()
            .lazy()
            .sort(["arrival_delay_m"], sorting_mode)
            .select([cols([
                "ID",
                "arrival_delay_m",
                "departure_delay_m",
                "city",
                "zip",
            ])])
            .limit(limit)
            .collect()
            .map_err(|e| {
                error!("could not query delay times: {}", e);
                StatusCode::BAD_REQUEST
            })?;

        df_to_json(delays)
    }
```

We get the `df_delays` data frame from the app state, clone it so we have an owned value, and call `.lazy()` to use the above mentioned [lazy API](https://docs.pola.rs/user-guide/lazy/).

Then we specify that we want the data sorted by the `arrival_delay_m` field, or the amount of minutes of delay, in the specified sorting order.

We also select a list of columns to return. Here, we could also use “*” to return all, or transformations to display the returned data in a different way.

Finally, we specify the limit of entries to return and call `collect()`, which executes the `LazyFrame` we’re building, giving us a returned `DataFrame`, or an error if something went wrong.

That’s it for our first query — not so difficult, right? If you have some experience with query languages such as SQL, most of this will be quite familiar.

To return the resulting data frame as JSON via the REST endpoint, we build a small helper function `df_to_json`:

```rust
    fn df_to_json(df: DataFrame) -> Result<Json<Value>, StatusCode> {
        serde_json::to_value(&df)
            .map_err(|e| {
                error!("could not serialize dataframe to json: {}", e);
                StatusCode::INTERNAL_SERVER_ERROR
            })
            .map(|v| v.into())
    }
```

With the `serde` feature, data frames are actually serializable, and we can just use `serde_json`. 

This means that they will be in a format specified by Polars, which is well suited for the purposes of data exploration but might not be for other purposes. So in those cases, another data transformation step would be necessary.

Next, let’s look at how we can filter data when implementing the endpoints for fetching population and data values for a given zip code.

## Filtering

We will implement both endpoints for fetching the population, as well as the area for a zip code at the same time, because they use the exact same underlying mechanism.

For this purpose, we’ll create a functin `cities_zip_to`:

```rust
    fn cities_zip_to(df_cities: &DataFrame, zip: &str, field: &str) -> Result<DataFrame, StatusCode> {
        df_cities
            .clone()
            .lazy()
            .filter(col("zip").str().contains_literal(lit(zip)))
            .select([col("zip"), col(field)])
            .collect()
            .map_err(|e| {
                error!("could not query {} for zip: {}", field, e);
                StatusCode::BAD_REQUEST
            })
    }
```

Here, we take the `df_cities` data frame, clone and lazify it, and then apply a filter operation. In this case, we want to filter on the `zip` column, making sure we only get results where this column contains the literal string of the `zip` value that’s being passed into the function.

To use the `str()` function here, we need the `strings` feature in Polars, and there is a whole [chapter](https://docs.pola.rs/user-guide/expressions/strings/) in the documentation on how to deal with strings.

Filters always have to evaluate to boolean types, and we can use multiple filters and even create our own custom filter functions.

Then we use `select` to get the `zip` column, as well as the column specified by the `field` string passed to the function. In our case, this will be either `pop` or `area`, depending on what we’re interested in.

With this helper function, the only thing we have to do is call it in both handler functions, which are structured in the same way as the `delays` handler:

```rust
    async fn cities_pop(
        Path(zip): Path<String>,
        app_state: State<Arc<AppState>>,
    ) -> Result<Json<Value>, StatusCode> {
        let pop_for_zip = cities_zip_to(&app_state.as_ref().df_cities, &zip, "pop")?;
        df_to_json(pop_for_zip)
    }

    async fn cities_area(
        Path(zip): Path<String>,
        app_state: State<Arc<AppState>>,
    ) -> Result<Json<Value>, StatusCode> {
        let area_for_zip = cities_zip_to(&app_state.as_ref().df_cities, &zip, "area")?;
        df_to_json(area_for_zip)
    }
```

One change from the previous handler is that we use the `Path` to get the `zip` code instead of the query parameters, but we could use either in this case.

That’s it for our filtering example. Let’s move on to joining both of our data frames by creating an endpoint for getting the amount of delays for a specified zip code.

## Using multiple data frames

Now, if you looked at the train delays data set in detail, you might have noticed that it already contains a zip code, so to get the delays of a certain zip code, we wouldn’t actually have to join the two data sets together.

However, to show off the functionality, we’ll do it anyway, and it’ll give us the benefit of being able to display the population of the zip codes together with the amount of delays that happened there, which might provide some insights while exploring the data.

So let’s look at how we can use our data sets joined by zip code, filter by zip code, group the results by this zip code, and show a nice summary of the data with the summed amount of delay minutes for a zip code:

```rust
    async fn delays_by_zip(
        Path(zip): Path<String>,
        app_state: State<Arc<AppState>>,
    ) -> Result<Json<Value>, StatusCode> {
        let delays_for_zip = app_state
            .as_ref()
            .df_delays
            .clone()
            .lazy()
            .join(
                app_state.as_ref().df_cities.clone().lazy(),
                [col("zip")],
                [col("zip")],
                JoinArgs::new(JoinType::Inner),
            )
            .group_by(["zip"])
            .agg([
                len().alias("len"),
                col("arrival_delay_m")
                    .cast(DataType::Int64)
                    .sum()
                    .alias("sum_arrival_delays"),
                col("arrival_delay_m").alias("max").max(),
                col("pop").unique().first(),
                col("city").unique().first(),
            ])
            .filter(col("zip").str().contains_literal(lit(zip.as_str())))
            .collect()
            .map_err(|e| {
                error!("could not query delay times by zip {}: {}", zip, e);
                StatusCode::BAD_REQUEST
            })?;

        df_to_json(delays_for_zip)
    }
```

The structure of the handler looks the same as before. We get the `zip` code via the `Path` and get the train delay data from the application state.

But then we use `join()` with our zip codes data set and join the two data sets on the `zip` column. If you look closely, you’ll notice that we use an `Inner` join. These join types are essentially the same you might know from a relational database, but Polars has a few extra, which you can check out [here](https://docs.pola.rs/user-guide/transformations/joins/).

In any case, here we are only interested in data points where both data sets have the corresponding zip code.

Then, we use `group_by` on the `zip` column, which aggregates all the values with the same zip code (in this case all, since we’re filtering for the zip code later down the query). Together with `group_by`, we use `.agg()` to aggregate data in different columns to create our resulting data output.

Here we specify a `len` column, which gives us the amount of rows which were grouped together for this result. Then, we sum up the `arrival_delay_m` to get the sum of delay minutes over all the delays of this zip code. 

We also calculate the maximum value and select the population and city for the delays, and since we group by zip, these will be the same for each row, and if we didn’t call `.unique()`, we’d get a list of duplicated populations and city names. So, we transform this into one singular value.

After that, we use the same filter that we used before for filtering for the zip code, and we’re done!

Those were quite a few transformations and expressions, but Polars DSL for building these, paired with the fantastic [documentation](https://docs.pola.rs/user-guide/expressions/) for expressions, makes this quite approachable.

## Determining linear correlations

Finally, let’s implement our last REST API endpoint. For this endpoint, we’d like to make it possible for the caller to specify two fields: `field1` and `field2` between which we calculate a correlation coefficient.

This coefficient can give us information on whether there is a linear relation between two sets of data. The result is between -1 and 1, and a value closer to 1 means that there is a stronger linear relation and vice versa.

Now, to properly calculate such a relation and make statistically significant statements on the data set, we’d have to handle missing data and properly make sure we have all values normalized and in a proper format.

However, for the purpose of this example, we’ll just calculate it using Polars on fields of our data set and see if we can figure out some interesting relations.

Polars provides multiple ways of doing this, but in this case, we’ll use the [Pearson Correlation Coefficient](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient) calculated by the [pearson_corr](https://docs.rs/polars-lazy/latest/polars_lazy/dsl/fn.pearson_corr.html) method.

So in our handler, we’ll join the two data sets together again so we can, for example, correlate something with the population and area as well, and then calculate the correlation coefficient for the given fields:

```rust
    async fn delays_corr(
        Path((field_1, field_2)): Path<(String, String)>,
        app_state: State<Arc<AppState>>,
    ) -> Result<Json<Value>, StatusCode> {
        let corr = app_state
            .as_ref()
            .df_delays
            .clone()
            .lazy()
            .join(
                app_state.as_ref().df_cities.clone().lazy(),
                [col("zip")],
                [col("zip")],
                JoinArgs::new(JoinType::Inner),
            )
            .select([
                len(),
                pearson_corr(col(&field_1), col(&field_2), 0)
                    .alias(&format!("corr_{}_{}", field_1, field_2)),
            ])
            .collect()
            .map_err(|e| {
                error!(
                    "could not query delay time correlation between {} and {}: {}",
                    field_1, field_2, e
                );
                StatusCode::BAD_REQUEST
            })?;

        df_to_json(corr)
    }
```

The query is similar to the previous endpoint, but we don’t group or aggregate. Rather, we simply select the length of the data used, which should be all rows, as a sanity check, and then we use the `pearson_corr` function on the given fields. 

If the fields don’t exist, or aren’t suitable for this type of check, we’ll get an error or an empty result (e.g. if we try to correlate a numeric and a text field).

The rest of the handler is the same as before, with getting our field values from the `Path` and returning a JSON result at the end.

That’s it for our queries and our REST API for exposing them — let’s test and see if what we created here actually works!

## Testing

We’ll start the server using the `RUST_LOG=info cargo run`  command and send requests to it using [curl](https://curl.se/).

Let’s start querying our first endpoint — the one that returns a number of sorted delays. First, without any parameters:

```bash
    curl "http://localhost:8000/delays"
```

And we get this response:

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "String",
          "name": "ID",
          "values": [
            "3699442958790718514-2407081448-7",
            "3699442958790718514-2407081448-9",
            ...
          ]
        },
        {
          "bit_settings": "SORTED_DSC",
          "datatype": "Int64",
          "name": "arrival_delay_m",
          "values": [
            159,
            157,
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "departure_delay_m",
          "values": [
            159,
            157,
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "String",
          "name": "city",
          "values": [
            "Essen",
            "Düsseldorf",
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "zip",
          "values": [
            45127,
            40210,
            ...
          ]
        }
      ]
    }
```

This is just the DataFrame serialized as JSON, and in a real world application, we might want to format the data in a different way as to make it easier to work with on the frontend, but that depends greatly on the use case.

Let’s parameterize the query and see if that works as well:

```bash
    curl "http://localhost:8000/delays?sorting=Asc&limit=5"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "String",
          "name": "ID",
          "values": [
            "7142163727799764498-2407122312-7",
            "-2290364085225859300-2407120928-11",
            ...
          ]
        },
        {
          "bit_settings": "SORTED_ASC",
          "datatype": "Int64",
          "name": "arrival_delay_m",
          "values": [
            0,
            0,
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "departure_delay_m",
          "values": [
            0,
            0,
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "String",
          "name": "city",
          "values": [
            "Bietigheim-Bissingen",
            "Ochsenfurt",
            ...
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "zip",
          "values": [
            74321,
            97199,
            ...
          ]
        }
      ]
    }
```

And we see that works as well — nice!

Let’s also see if our zip-based queries work.

First, the query to find out the population for a specific zip code:

```bash
    curl "http://localhost:8000/cities/52062/population"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "zip",
          "values": [
            52062
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "pop",
          "values": [
            15989
          ]
        }
      ]
    }
```

Great, that works. Let’s try the same for the area:

```bash
    curl "http://localhost:8000/cities/52062/area"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "zip",
          "values": [
            52062
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Float64",
          "name": "area",
          "values": [
            1.639112
          ]
        }
      ]
    }
```

Perfect, we have 15,989 people living in an area of 1,639 km², which seems correct based on the [data origin](https://www.suche-postleitzahl.org/plz-gebiet/52062).

Next, let’s test querying both data sets, getting the number of delays for a specific zip code:

```bash
    curl "http://localhost:8000/delays/80331/"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "zip",
          "values": [
            80331
          ]
        },
        {
          "bit_settings": "",
          "datatype": "UInt32",
          "name": "len",
          "values": [
            22358
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "sum_arrival_delays",
          "values": [
            70352
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Int64",
          "name": "max",
          "values": [
            36
          ]
        },
        {
          "bit_settings": "SORTED_ASC",
          "datatype": "Int64",
          "name": "pop",
          "values": [
            4741
          ]
        },
        {
          "bit_settings": "SORTED_ASC",
          "datatype": "String",
          "name": "city",
          "values": [
            "München"
          ]
        }
      ]
    }
```

We can see that at “80331” in Munich, there were 22,358 delays, summing up to 70,352 minutes, with a maximum singular delay of 36 minutes. Cool!

Finally, let’s see if our endpoint for determining correlations works. First, let’s examine if there is a correlation between the delay at arrival and the delay at departure. We would definitely expect to find a high correlation there. (Above-mentioned caveats apply):

```bash
    curl "http://localhost:8000/delays/corr/arrival_delay_m/departure_delay_m/"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "UInt32",
          "name": "len",
          "values": [
            2050194
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Float64",
          "name": "corr_arrival_delay_m_departure_delay_m",
          "values": [
            0.9555141639554701
          ]
        }
      ]
    }
```

And sure enough, we find a correlation coefficient of `0.956`, which points to a strong linear correlation. We can also see that all two million or so entries were used for this query.

Let’s see if there is a correlation between the arrival delay and the amount of people living at a zip code:

```bash
    curl "http://localhost:8000/delays/corr/arrival_delay_m/pop/"
```

```json
    {
      "columns": [
        {
          "bit_settings": "",
          "datatype": "UInt32",
          "name": "len",
          "values": [
            2050194
          ]
        },
        {
          "bit_settings": "",
          "datatype": "Float64",
          "name": "corr_pop_arrival_delay_m",
          "values": [
            0.0049400151164128055
          ]
        }
      ]
    }
```

Nope, doesn’t seem like it based on our data set. While in big cities there will likely be more delays, because there are simply more trains (and maybe other factors), the same doesn’t hold for singular zip codes.

But again, at this level, we’re essentially still playing with the data and not finding any statistically significant results.

In any case, it seems our implementation works, and it’s now possible to explore and play around with the data set via REST calls, which was the goal. Woohoo!

You can find the full code for this example on [GitHub](https://github.com/zupzup/rust-polars-example) (this doesn’t include the data files — you’ll have to fetch them on your own from the links mentioned below).

## Polars concepts

One of the main selling points of Polars over similar solutions such as [Pandas](https://pandas.pydata.org/) is performance. Polars is written in highly optimized Rust and uses the [Apache Arrow](https://arrow.apache.org/) container format.

Besides the Rust API, Polars also exposes a Python API, but in this article, we only focused on using the Rust API to check out Polars’ data analysis capabilities.

Now, if you’ve never done any data analysis work, you might ask “Why would I use this at all?” Well, data is everywhere, and almost regardless of what you do, taking a look at the data around the thing you do and analyzing it from different perspectives will likely give you interesting insights into how to improve things.

For example, if you collect usage data for a frontend application, or performance data, event logs, errors logs, or really anything you can imagine, there are likely treasure troves of insights hidden into these data sets.

You can then analyze those same datasets using a tool such as Polars. This could result in insights that help improve user flows and increase traffic, engagement, and conversion.

Another use case would be to expose an existing data set and make it available to a frontend application with advanced querying capabilities so that the application can leverage the data.

But first, let’s learn a little more about Polars.

As mentioned above, Polars is a dataframe interface library. So what’s a dataframe? Essentially, it’s just data laid out in a two-dimensional way with rows and columns. Polars is a tool that enables us to query such data frames with very high performance and make it as easy as possible to formulate these, at times, very complex queries.

If you want to get into Polars, the library is very well documented, and I’d recommend you check out their [getting started](https://docs.pola.rs/) tutorial, their API [docs](https://docs.rs/polars/latest/polars/index.html), and when you’re all set up, you can also check out their [Cookbooks](https://docs.rs/polars/latest/polars/index.html#cookbooks) to learn about many of the standard operations within Polars.

When you start working with Polars, the first step is to import data into the application, and there are multiple ways to do so, depending on the data format you’re working with. Polars supports [several data formats](https://docs.pola.rs/user-guide/io/), and it’ll depend on whether you want to load all the data at once or lazily import the data as needed.

Once the data is imported, you have your first `DataFrame`, although you can also use the nifty `df!` macro to manually create a data frame for playing around.

The next step is to inspect the data by printing it, or rather printing a part of it using the `head` or `tail` functions. If you have a wide data frame, you won’t see all the data in the printed out form, but you can increase the amount of printed columns using the `POLARS_FMT_MAX_COLS=100` environment variable.

Upon inspecting your data, you will notice, that different columns have different [data types](https://docs.pola.rs/user-guide/concepts/data-types/overview/).  When you want to operate on data, you will encounter the concept of [contexts](https://docs.pola.rs/user-guide/concepts/contexts/), such as `select`, `filter`, and `group_by`.

These contexts work together with [expressions](https://docs.pola.rs/user-guide/expressions/) (such as `col`, `sort`, `sum`, `unique` etc.) to orchestrate queries. You can also create custom expressions, but there is already a large library of common ones available, which should be enough for most use cases. 

Getting familiar with the different expressions and how to orchestrate them to get results is one of the main learning areas when starting to use Polars for data analysis.

Beyond that, there is the concept of [transformations](https://docs.pola.rs/user-guide/transformations/), which has to do with merging data frames together, or transforming a data frame to a differently structured data frame.

Another important distinction in Polars is the one between eager and lazy evaluation. Essentially, you can create a query lazily and then execute it on-demand, which means certain optimizations can be run that can’t be run otherwise.

Or, you can run it eagerly, which executes immediately. The trade offs are discussed in the [docs](https://docs.pola.rs/user-guide/concepts/lazy-vs-eager/), but from my perspective, it almost always makes sense to use the lazy API, as you can execute the lazy queries in an eager fashion as well.

## Conclusion

In this article, we took a look at [Polars](https://pola.rs/) and how to use it with Rust for high-performance data analysis tasks. We also checked out how we can expose this data analysis capability via a REST API so it can be explored and consumed by frontend web clients.

Polars has made immense progress over the last few years and is on a trajectory to be one of the most widely used and best performing libraries for data analysis. Due to its fantastic documentation and similarity in API to other solutions in the space, the barrier to entry is low, and one can start getting their hands dirty with data shortly after installation.

