---
title: "Get Enterprisey with Rust"
date: 2022-05-23T17:33:09+02:00
draft: false
---
Why not use the coolest language out there to do the things you probably still use Java for? Rust is marketed as a 'systems language', whatever that is. It looks to me like a general purpose language, 'turing complete' and whatnot. There are plenty crates for anything web related. There's tools for http servers, database connections, logging. We may also want security, monitoring, telemetry, cloud deployment. Do we want end-to-end testing? Sure. Openapi? Love it. 

That said, a framework like spring-boot is pretty mature. It may just be a hassle trying to accomplish those nice features...

### Challenge accepted...

Start a new project 
{{<highlight bash>}}
cargo new hello-web
{{</highlight>}}

## Web framework.

There are three crates on the shortlist
- [Actix-web](https://crates.io/crates/actix-web)
- [Warp](https://crates.io/crates/warp)
- [Axum](https://crates.io/crates/axum)

This is not going to be an exhaustive comparison. I chose Axum based on [this](https://kerkour.com/rust-web-framework-2022). And I like the fact that its from the developers of [Tokio](https://tokio.rs/), let's say the [netty](https://netty.io/) of the rust world.

A hello world from Axum (found [here](https://github.com/tokio-rs/axum/tree/main/examples/hello-world))

{{<highlight rust "linenos=table">}}
use axum::{response::Html, routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    
    println!("listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
{{</highlight>}}

That's pretty straightforward. It'll get more complicated than this, but this code is on par with any webframework out there.

```Cargo.toml
axum = "0.5.6"
```

## Restartability

Haven't you added this to your -ilities? Still restarting manually? You shouldn't have to! 
It became a must-have for web development, and of course it is available for java as well, but al too often it is lacking from, what have you, the average maven project (or does it simply take too long?)

Luckily for us, using a cargo plugin this is a breeze:
{{<highlight bash "linenos=table">}}
cargo install cargo-watch
cargo watch -x run
{{</highlight>}}

## Logging

There are 3 options (probably more): 
* [env_logger](https://crates.io/crates/env_logger): simple to set up
* [log4rs](https://crates.io/crates/log4rs), more advanced use cases, modelled after the now infamous log4j
* [tracing](https://crates.io/crates/tracing), 'tracing is a framework for instrumenting Rust programs to collect structured, event-based diagnostic information.'

Useful comparison [here](https://medium.com/nerd-for-tech/logging-in-rust-e529c241f92e)

Here I would have opted for env_logger, because we have a simple project, but I ended up with tracing, because it integrates really well with tokio and axum. It's also very well suited for larger environments and other subscribers to logging events than mere log files. Read [here](https://burgers.io/custom-logging-in-rust-using-tracing) if you want to know more.

```Cargo.toml
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

Add a simple tracing 'subscriber' to your code. It will receive log events and write them to stdout.

{{<highlight rust "linenos=table">}}
use tracing::{debug,Level};
use tracing_subscriber::FmtSubscriber;

let subscriber = FmtSubscriber::builder()
        .with_max_level(Level::TRACE)
        .finish();
tracing::subscriber::set_global_default(subscriber)
        .expect("setting default subscriber failed");
{{</highlight>}}

## Database

We'll just use postgresql. Why not with podman?
```bash
podman machine init --cpus=1 --disk-size=10 --memory=2048 -v $HOME:$HOME
podman machine start
podman run -dt --name my-postgres -e POSTGRES_PASSWORD=1234 -v "postgres:/var/lib/postgresql/data:Z" -p 5432:5432 postgres
```
Article [here](https://medium.com/@butkovic/favoring-podman-over-docker-desktop-33368e031ba0)

### Connect to it

Of course we need to connect to the database from our program. For this we'll add 
```Cargo.toml
sqlx = { version = "0.5.13", features = ["postgres", "runtime-tokio-native-tls"] }
```

and
{{<highlight rust "linenos=table">}}
use sqlx::postgres::{PgPool, PgPoolOptions};

let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");
{{</highlight>}}

For this I looked at the [sqlx example](https://github.com/tokio-rs/axum/tree/main/examples/sqlx-postgres) for Axum. [Sqlx](https://crates.io/crates/sqlx) is a database driver for several databases, one of which is postgres. They state: _SQLx is not an ORM!_. There are ORM's built on top of Sqlx, but we're not going to use them.

### Setting up the schema

We're going to emulate spring-boot here: on server startup recreate the database using a script with ddl and dml statements. Not for production, but fine for local testing.

{{<highlight rust "linenos=table">}}
let create_database_sql = include_str!("create_database.sql");
let statements = create_database_sql.split(";");
for statement in statements {
    sqlx::query(statement).execute(&pool).await.expect("error running script");
}
{{</highlight>}}


Add _create_database.sql_ to _src_:
{{<highlight sql "linenos=table">}}
drop table if exists blog_entry;
create table blog_entry
(
    title   varchar(100),
    author  varchar(40),
    text    text
);

insert into blog_entry(created, title, author, text)
values ('Get enterprisey with Rust', 'Sander', 'Lorem Ipsum');
insert into blog_entry(created, title, author, text)
values ('Get whimsical with data', 'Sander', 'Lorem Ipsum');
{{</highlight>}}

Note that ```include_str``` is a macro that reads a file that is part of the compilation unit, similar to reading a resource from the java classpath. Like in JDBC you have to split the file into individual statements and then execute them.

### Create a service that returns Json

[Serde](https://crates.io/crates/serde) is the default for working with Json, so add ```serde = "1.0.137"``` to your Cargo.toml. 
Next we create an object that can be serialized:

{{<highlight rust>}}
#[derive(Serialize, Deserialize, Clone, Debug, sqlx::FromRow)]
struct BlogEntry{
    title: String,
    author: String,
    text: String
}
{{</highlight>}}

As you can see a whole lot of derived traits are being used. 
* ```Serialized``` and ```Deserialized``` for translating from and to Json.
* ```Clone``` and ```Debug``` are not stricly necessary, but come in handy.
* ```sqlx::FromRow``` automagically maps the database result to the desired type.

Now we have to create a function that executes the query and hands back the result to the client.

{{<highlight rust "linenos=table">}}
use axum::{extract::Extension, http::StatusCode, Json, Router, routing::get};

async fn get_blogs(Extension(pool): Extension<PgPool>) -> Result<Json<Vec<BlogEntry>>, (StatusCode, String)> {
    debug!("handling request");

    sqlx::query_as("select title, author, text from blog_entry")
        .fetch_all(&pool)
        .await
        .map(|r| Json(r))
        .map_err(internal_error)
}
{{</highlight>}}

* async function
* Not the peculiar syntax ```Extension(pool): Extension<PgPool>```. This is pattern matching on function arguments. The actual argument will be passed by Axum. We only need the pool and this way we can extract it from the Extension.
* For Json you need to wrap the result ```Vec<BlogEntry>``` in a ```axum::Json``` struct. 
* ```map_err``` is called with function argument ```internal_error```. This function maps any runtime error to http code 500.

{{<highlight rust "linenos=table">}}
fn internal_error<E>(err: E) -> (StatusCode, String)
    where
        E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
{{</highlight>}}

Now we need to set up the server so that a ```/entries``` route is mapped to our newly created function. In ```main``` remove the _hello world_ handler and add this:

{{<highlight rust "linenos=table">}}
let app = Router::new()
    .route("/entries", get(get_blogs))
    .layer(Extension(pool));

let addr = SocketAddr::from(([127, 0, 0, 1], 3000));

debug!("listening on {}", addr);

axum::Server::bind(&addr)
    .serve(app.into_make_service())
    .await
    .unwrap();
{{</highlight>}}

### Working with dates
Dates are indispensible in pretty much any enterprisey application. For this we will add the ```chrono``` crate. The rust standard library is not as comprehensive as in some other languages. This means that there is no formal standard for working with databases, or dates, but in many cases (serde, chrono, tokio) there are _de facto_ standards. This means that there are also implementations in serde and sqlx for working with chrono. For them to work we need to add _features_ for both in the Cargo.toml:
```Cargo.toml
sqlx = { version = "0.5.13", features = ["postgres", "runtime-tokio-native-tls", "chrono"] }
chrono = {version = "0.4", features = ["serde"]}
```
Or rather add _chrono_ feature to sqlx and add _serde_ to _chrono_. I don't really know why this is not symmetrical. At least it works.

We add a _created_ date type to the BlogEntry struct:
{{<highlight rust>}}
use chrono::{DateTime, Utc};

#[derive(Serialize, Deserialize, Clone, Debug, sqlx::FromRow)]
struct BlogEntry {
    created: DateTime<Utc>,
    //...
}
{{</highlight>}}

and add a column to _create_database.sql_

{{<highlight sql>}}
create table blog_entry
(
    created timestamptz,
    //...
);

insert into blog_entry(created, title, author, text)
values (now(), 'Get enterprisey with Rust', 'Sander', 'Lorem Ipsum');
insert into blog_entry(created, title, author, text)
values (now(), 'Get whimsical with data', 'Sander', 'Lorem Ipsum');
{{</highlight>}}

We now have a http rest service on port 3000 that serves blog entries from a postgres database. 
```bash
$ curl http://localhost:3000/entries
[{"created":"2022-05-27T06:45:27.750171Z","title":"Get enterprisey with Rust","author":"Sander","text":"Lorem Ipsum"},{"created":"2022-05-27T06:45:27.756820Z","title":"Get whimsical with data","author":"Sander","text":"Lorem Ipsum"}]
%
```
Neat!

Now, this is not production ready. We need automated tests, security, what about monitoring or performance metrics?
OpenAPI implementation is also on my whishlist.
Also what about database migrations? At this point I honestly don't know if there is like _flyway_ for rust. I'll have to get back to you on this.

The full code so far is on https://github.com/shautvast/rust-rest-service.

...to be continued