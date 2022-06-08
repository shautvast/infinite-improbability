---
title: "Get Enterprisey with Rust part 3 - Environments"
date: 2022-06-01T17:28:32+02:00
draft: false
---
This is the third installment in our series about dealing with enterprisey needs using rust. 
* [part 1](/enterprisey) was about the initial setup with axum and postgres, logging and dates.
* [part 2](/enterprisey) was devoted to input validation on incoming Json objects.

### Reading environment variables

I realized that the code I put in for part 1 on github already uses the built-in ```std::env::var``` function, but I didn't mention it in the blog.

{{<highlight rust "linenos=table">}}
let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:1234@localhost".to_string());
{{</highlight>}}

The ```std::env::var``` function returns a ```Result``` that either contains the environment variable, or in the case that it is not found, the default, supplied in ```unwrap_or_else```.

Check out more [documentation](https://doc.rust-lang.org/book/ch12-05-working-with-environment-variables.html). 

### Dotenv

[Dotenv](https://crates.io/crates/dotenv) is another option. Upon executing ```dotenv().ok()``` it reads a file with called _.env_ (in the current directory, or any parent) and puts the contents in your environment as variables. This might be more convient depending on your circumstances.

```Cargo.toml
dotenv = "0.15.0"
```

The code is now:
{{<highlight rust "linenos=table">}}
dotenv::dotenv().ok();

let db_connection_str = std::env::var("DATABASE_URL").unwrap();
{{</highlight>}}

Note that the ```unwrap``` function will abort the program, in case the DATABASE_URL is not found, but I also added _.env_ to the project root, so the program will run just fine (using cargo in the project directory). Of course, when deployed you need to make sure the file can be found and you must supply a environment specific version of it (to the container).
```
DATABASE_URL=postgres://postgres:1234@localhost
```

Dotenv also allows variable substitution within _.env_ files, allowing for less duplication.