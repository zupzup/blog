In this post we'll take a look at how to do database migrations using the [refinery](https://github.com/rust-db/refinery) library in the context of a web application.

For the web application, we'll use [warp](https://github.com/seanmonstar/warp) and as a database we'll use postgres.

The scenarios we'll cover in this post are adding a table, adding a field and creating and migrating some data.

## Implementation

First, let's spin up a basic warp server with a health endpoint:

```rust
type Result<T> = std::result::Result<T, Rejection>;
type Error = Box<dyn std::error::Error + Send + Sync + 'static>;

#[tokio::main]
async fn main() {
    run_migrations().await.expect("can run DB migrations: {}");

    let health_route = warp::path!("health").and_then(health_handler);

    let routes = health_route.with(warp::cors().allow_any_origin());

    println!("Started server at localhost:8000");
    warp::serve(routes).run(([0, 0, 0, 0], 8000)).await;
}

async fn health_handler() -> Result<impl Reply> {
    Ok(StatusCode::OK)
}
```

Alright, starting this will run a web server on port 8000 and we can check `http://localhost:8000/health` to see if it's up.

In order to configure `refinery`, we need to use the `embed_migrations` macro, telling the library where we store the sql migrations:

```rust
mod embedded {
    use refinery::embed_migrations;
    embed_migrations!("migrations");
}
```

So far, so good. Next, we'll add some database migrations in the `migrations` folder:

```sql
// V1__initial.sql
CREATE TABLE IF NOT EXISTS todo
(
    id SERIAL PRIMARY KEY NOT NULL,
    name VARCHAR(255),
    created_at timestamp with time zone DEFAULT (now() at time zone 'utc'),
    checked boolean DEFAULT false
);
```
 and 

```sql
// V2__add_checked_date.sql
ALTER TABLE todo
    ADD COLUMN checked_date timestamp with time zone;
```

In the first, initial migration, we add our `todo` table and in the second one, we add a field called `checked_date`, which indicates when a certain todo has been checked off the list.

Now, let's see how to actually run these migrations on server startup:

```rust
async fn run_migrations() -> std::result::Result<(), Error> {
    println!("Running DB migrations...");
    let (mut client, con) = tokio_postgres::connect("host=localhost user=postgres", NoTls).await?;

    tokio::spawn(async move {
        if let Err(e) = con.await {
            eprintln!("connection error: {}", e);
        }
    });
    let migration_report = embedded::migrations::runner()
        .run_async(&mut client)
        .await?;

    for migration in migration_report.applied_migrations() {
        println!(
            "Migration Applied -  Name: {}, Version: {}",
            migration.name(),
            migration.version()
        );
    }

    println!("DB migrations finished!");

    Ok(())
}
```

Since we use the async `tokio-postgres`, we first create a connection with that. Then we run the migrations using the `refinery` migrations runner and use the returned `Report` to print information about the applied migrations, logging the migration name and version of each applied migration.

Let's now try it with our migrations. First, the two mentioned above.

When we run the server using `cargo run`, we get the following log output:

```bash
Running DB migrations...
Migration Applied -  Name: initial, Version: 1
Migration Applied -  Name: add_checked_date, Version: 2
DB migrations finished!
```

Nice, it seems to have worked. If we check the database, we'll see our `todo` table with a `checked_date` field added to it.

Let's add two more migrations. First, adding a few Todos:

```sql
// V3__add_data.sql
INSERT INTO todo (name) VALUES ('pet my cat');
INSERT INTO todo (name) VALUES ('pet my dog');
INSERT INTO todo (name) VALUES ('feed my pets');
```

... and a very simplistic data migration. We'll just check off all the created Todos:

```sql
// V4__migrate_data.sql
UPDATE todo SET checked = true, checked_date = now(); 
```

And we run the server again:

```bash
Running DB migrations...
Migration Applied -  Name: add_data, Version: 3
Migration Applied -  Name: migrate_data, Version: 4
DB migrations finished!
```

If you check the database, you'll see some todos in there now, which have already been checked off, showing us that it worked nicely. :)

The full example code can be found [here](https://github.com/zupzup/refinery-warp-example)

## Conclusion

Automatic DB migrations are a very useful feature in services, where the database schema changes regularly and where it's possible that complex migration operations are needed at some point.

The `refinery` library provides everything you'd want and is very well documented, so it would currently be my first choice in this case. :)

#### Resources

* [Code Example](https://github.com/zupzup/refinery-warp-example)
* [refinery](https://github.com/rust-db/refinery)
