As sad as it is, sometimes as a developer, you're asked to implement an Excel export for customers. Why anyone would need, or use such a thing instead of CSV is beyond me, but after more than 10 years in this field, this is something that just pops up.

Luckily, there is the [xlsxwriter](https://github.com/informationsea/xlsxwriter-rs) crate in the Rust ecosystem, which provides bindings to the widely used [libxlsxwriter](https://github.com/jmcnamara/libxlsxwriter) C library.

In this post, we will take a look at how to create custom Excel reports from a Rust web service. We will use some custom data types, formatting and it will still be fast and relatively lean in memory consumption, even for large files.

Let's get started!

## Implementation

First, let's start by building a basic web server with a route pointing to our excel export.

Let's start with dependencies:

```yaml
[dependencies]
tokio = { version = "0.2.21", features = ["macros", "rt-threaded", "blocking", "time"] }
warp = "0.2.3"
thiserror = "1.0.20"
chrono = { version = "0.4.13", features = ["serde"] }
xlsxwriter = "0.3.1"
uuid = { version = "0.8", features = ["v4"] }
lazy_static = "=1.4.0"
```

Since we're building a Warp web app, based on tokio, the first couple of dependencies aren't that surprising.

We also add `chrono`, to show off how to handle dates in Excel and we'll use `uuid` to create some random ids.

The next step is to create a basic, runnable web server with a report endpoint that doesn't do anything yet:

```rust
#[tokio::main]
async fn main() {
    let report_route = warp::path("report")
        .and(warp::get())
        .and_then(report_handler);

    println!("Server started at localhost:8080");
    warp::serve(report_route).run(([0, 0, 0, 0], 8080)).await;
}

async fn report_handler() -> Result<impl Reply> {
    Ok("report endpoint")
}
```

Great. The next step is to define some test data we want to export to excel. In practice, we would fetch this from a database, or from other web services, but here we will just generate some random data.

To do this, we'll define the `Thing` data structure and generate some random data for it:

```rust
lazy_static! {
    static ref THINGS: Vec<Thing> = create_things();
}

#[derive(Clone, Debug)]
pub struct Thing {
    pub id: String,
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub project: String,
    pub name: String,
    pub text: String,
}

fn create_things() -> Vec<Thing> {
    let mut result: Vec<Thing> = vec![];
    for _ in 0..1000 {
        result.push(Thing {
            id: random_string(),
            start_date: Utc::now(),
            end_date: Utc::now(),
            project: random_string(),
            name: random_string(),
            text: random_string(),
        });
    }
    result
}

fn random_string() -> String {
    Uuid::new_v4().to_string()
}
```

Essentially, every `Thing` has some string fields and some date fields. The date fields we set to `now()`, because we're only interested in the formatting of the dates. The rest of the fields are filled with UUIDs. Also, we're making this initialized data available globally using `lazy_static`. That way we only have to create it once at startup.

With that out of the way, we can implement our `report_handler` properly:

```rust
async fn report_handler() -> Result<impl Reply> {
    let now = Instant::now();
    let result = tokio::task::spawn_blocking(move || excel::create_xlsx(THINGS.to_vec()))
        .await
        .expect("can create result");
    println!("report took: {:?}", now.elapsed());
    Ok(result)
}
```

Since we're also interested in how long creating a huge Excel file takes, we're going to note down the time the generation took. The actual Excel generation will be implemented in the `excel::create_xlsx` function in the next part of this post.

We use Tokio's `spawn_blocking` function, which is only available when using the `blocking` feature, to spawn the Excel creation on the blocking thread pool within Tokio. This is useful in asynchronous code bases, when you have a blocking workload (e.g. creating a huge XLSX file ;)).

If you don't spawn this on the `blocking` thread pool, the scheduler threads of the async runtime might get blocked instead, which will slow down the whole application.

Let's look at how to actually create the XLSX file next.

First, in the `excel.rs` module, let's create the `create_xlsx` function:

```rust
use xlsxwriter::{DateTime as XLSDateTime, Format, Workbook, Worksheet};

const FONT_SIZE: f64 = 12.0;

pub fn create_xlsx(values: Vec<Thing>) -> Vec<u8> {
    let uuid = Uuid::new_v4().to_string();
    let workbook = Workbook::new(&uuid);
    let mut sheet = workbook.add_worksheet(None).expect("can add sheet");
    ...
```

The first step is to create a new workbook and, within that workbook, add a new sheet. We will only use one sheet in this example, but you could add more if you wanted to.

We create a uuid as the name of the workbook, because that's the file name it will be saved under and we want that to be unique, so it won't be overwritten by a call at the same time.

Next, we will deal with the problem of `width` in excel. The issue is, that we could just write our values in the rows and columns we wanted, but if the text, for example, is longer than the default width of the cell, it won't look nice and the user will have to increase the size to read the content.

Optimally, we would like to make the cells big enough for the whole text we put in to be visible, up to a certain point. In order to do this, however, we need to keep track of the maximum value length within every column, so that we can adjust the width of these cells afterwards. You might ask yourself why there isn't a "auto-width"-feature in the API. That's because that's a runtime-only feature of Excel and can't be written to the file itself. So we need to do it by hand.

For this purpose, we will use a concept called a `width_map`:

```rust
    let mut width_map: HashMap<u16, usize> = HashMap::new();
```

This is simply a mapping from column number to width in character length of the included value. At the end, we will multiply the width by 1.2 to actually set the width. This is because it's not trivial to know the exact pixel width based on the font and font size we're using. This might seem hacky, but it works well and is actually what was [recommended](https://github.com/jmcnamara/libxlsxwriter/issues/118) by the libxlsxwriter author as well as a simple workaround. A more fancy version would be to map each character to it's exact pixel width and calculate the exact width that way.

Next, we're going to create the header row:

```rust
    create_headers(&mut sheet, &mut width_map);
```

...which looks like this:

```rust
fn create_headers(sheet: &mut Worksheet, mut width_map: &mut HashMap<u16, usize>) {
    let _ = sheet.write_string(0, 0, "Id", None);
    let _ = sheet.write_string(0, 1, "StartDate", None);
    let _ = sheet.write_string(0, 2, "EndDate", None);
    let _ = sheet.write_string(0, 3, "Project", None);
    let _ = sheet.write_string(0, 4, "Name", None);
    let _ = sheet.write_string(0, 5, "Text", None);

    set_new_max_width(0, "Id".len(), &mut width_map);
    set_new_max_width(1, "StartDate".len(), &mut width_map);
    set_new_max_width(2, "EndDate".len(), &mut width_map);
    set_new_max_width(3, "Project".len(), &mut width_map);
    set_new_max_width(4, "Name".len(), &mut width_map);
    set_new_max_width(5, "Text".len(), &mut width_map);
}
```

Essentially, this is just boilerplate, which adds a column for each of our data points in the first (0th) row, with the title.

Here, we also use the `set_new_max_width` function with our `width_map`:

```rust
fn set_new_max_width(col: u16, new: usize, width_map: &mut HashMap<u16, usize>) {
    match width_map.get(&col) {
        Some(max) => {
            if new > *max {
                width_map.insert(col, new);
            }
        }
        None => {
            width_map.insert(col, new);
        }
    };
}
```

As you can see, for each column, we go ahead and update the maximum width. In this case, we simply take the string length of our hardcoded header values. But we will use the same technique later on with our generated random data.

Before we get to that though, let's define some formatting rules back in the `create_xlsx` function:

```rust
    let fmt = workbook
        .add_format()
        .set_text_wrap()
        .set_font_size(FONT_SIZE);

    let date_fmt = workbook
        .add_format()
        .set_num_format("dd/mm/yyyy hh:mm:ss AM/PM")
        .set_font_size(FONT_SIZE);
```

These two formats are just for basic text and for dates. It's useful to define them once, so we don't have to allocate one for each row/column.

With that out of the way, we can go ahead and add one row for each of our `Things`:

```rust
    for (i, v) in values.iter().enumerate() {
        add_row(i as u32, &v, &mut sheet, &date_fmt, &mut width_map);
    }
```

```rust
fn add_row(
    row: u32,
    thing: &Thing,
    sheet: &mut Worksheet,
    date_fmt: &Format,
    width_map: &mut HashMap<u16, usize>,
) {
    add_string_column(row, 0, &thing.id, sheet, width_map);
    add_date_column(row, 1, &thing.start_date, sheet, width_map, date_fmt);
    add_date_column(row, 2, &thing.end_date, sheet, width_map, date_fmt);
    add_string_column(row, 3, &thing.project, sheet, width_map);
    add_string_column(row, 4, &thing.name, sheet, width_map);
    add_string_column(row, 5, &thing.text, sheet, width_map);

    let _ = sheet.set_row(row, FONT_SIZE, None);
}
```

The `add_row` function simply goes over all the fields we want to display and calls helper functions for setting string and date fields. Then, at the end, we set the height of the row to the `FONT_SIZE` property, to ensure we don't cut off the text vertically.

Let's look at the helpers next:

```rust
fn add_string_column(
    row: u32,
    column: u16,
    data: &str,
    sheet: &mut Worksheet,
    mut width_map: &mut HashMap<u16, usize>,
) {
    let _ = sheet.write_string(row + 1, column, data, None);
    set_new_max_width(column, data.len(), &mut width_map);
}

fn add_date_column(
    row: u32,
    column: u16,
    date: &DateTime<Utc>,
    sheet: &mut Worksheet,
    mut width_map: &mut HashMap<u16, usize>,
    date_fmt: &Format,
) {
    let d = XLSDateTime::new(
        date.year() as i16,
        date.month() as i8,
        date.day() as i8,
        date.hour() as i8,
        date.minute() as i8,
        date.second() as f64,
    );

    let _ = sheet.write_datetime(row + 1, column, &d, Some(date_fmt));
    set_new_max_width(column, 26, &mut width_map);
}
```

The string case is rather simple. Essentially, we just set the string with the format for strings using `write_string` and update the `width_map` with the string width of the data. Setting a date is a bit more involved, because we need to crate a `libxlsxwriter::DateTime` from our `chrono::DateTime` first.

But once that is done, the rest is exactly the same, with the exception that we manually set the width to `26`, which is the string width of the date format we're using and that we use `write_datetime`.

With the rows set, the last step is to actually set the width of each column to our calculated max values, generate the worksheet and return it to the caller:

```rust
    width_map.iter().for_each(|(k, v)| {
        let _ = sheet.set_column(*k as u16, *k as u16, *v as f64 * 1.2, Some(&fmt));
    });

    workbook.close().expect("workbook can be closed");

    let result = fs::read(&uuid).expect("can read file");
    remove_file(&uuid).expect("can delete file");
    result
```

We iterate the `width_map` and for each column in there, set the width of the column to 1.2 times the max width, which should make it so we can comfortably read everything in the sheet.

Then we use `workbook.close()` to write the excel file and read it in, so we can return it to the caller. There doesn't seem to be a way to simply return the bytes from `libxlsxwriter`, at least I didn't find a way, but that's OK.

After reading the file into memory, we delete it, so as to not fill up the disk and return it to the handler.

We can now run our app with `cargo run` and test it by calling `curl http://localhost:8080/report > rep.xlsx`.

Upon opening `rep.xlsx` with either Excel, or LibreOffice (or whatever your favorite tool is for this :)), you should see a well-formed Excel file, with widths and heights set according to the content and font size. The dates should be actual excel dates as well.

```bash
report took: 76.507859ms
```

<center>
    <a href="images/report.png" target="_blank"><img src="images/report_thmb.png" /></a>
</center>


It works!

The full example code can be found [here](https://github.com/zupzup/xlsx-in-rust-example)

## Conclusion

While supporting xlsx is likely not the dream of many developers, it's something that comes up here and there and fortunately, due to Rust's fantastic C interoperability and the great `libxlsxwriter` library, we were able to deal with it quite nicely.

In my tests the performance of the outlined approach in terms of execution time and memory footprint was far and away better than even a streaming implementation using [Apache POI](https://poi.apache.org/), but your mileage may vary depending on your use-case and setup. It's also not a fair comparison, as POI has a huge set of features far beyond libxlsxwriter and isn't necessarily optimized for performance, but rather for completeness.

#### Resources

* [Code Example](https://github.com/zupzup/xlsx-in-rust-example)
* [libxlsxwriter](https://github.com/jmcnamara/libxlsxwriter)
* [Warp](https://github.com/seanmonstar/warp)
