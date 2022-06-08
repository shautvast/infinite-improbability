---
title: "Get Enterprisey with Rust part 2 - Input Validation"
date: 2022-05-30T17:50:11+02:00
author: Sander Hautvast
draft: false
---
Input validation is next on the list of enterprisey features (see also [part 1](/enterprisey)). This will be the only new feature to add in this post because we need to add quite a but of boilerplate (sadly) to get it working.

First we need to add a service that will respond to a post request on the same url. 

Our app definition now looks like this:
{{<highlight rust "linenos=table">}}
let app = Router::new()
        .route("/entries", get(get_blogs).post(add_blog))
        .layer(Extension(pool));
{{</highlight>}}

And this is the ```add_blog``` function:

{{<highlight rust "linenos=table">}}
async fn add_blog(Extension(pool): Extension<PgPool>, ValidatedJson(blog): ValidatedJson<BlogEntry>) -> Result<Json<String>, (StatusCode, String)> {
    debug!("handling BlogEntries request");

    sqlx::query("insert into blog_entry (created, title, author, text) values ($1, $2, $3, $4)")
        .bind(blog.created)
        .bind(blog.title)
        .bind(blog.author)
        .bind(blog.text)
        .execute(&pool)
        .await
        .map_err(internal_error)?;

    Ok(Json("created".to_owned()))
}
{{</highlight>}}

This won't compile for now, because we still have to define ValidatedJson.

Here we see a different form of working with SQLx. We now have bind parameters. O right, that's a must have for performance and security! Note that there is also a macro called query!. The main advantage of this is that it does compile-time query verification! I did not include it here for simplicity. It needs a environment variable in the shell where you run Cargo.

Now, you could also have used axum::Json instead of the custom ValidatedJson type. It would try to deserialize the json String and raise an error if for instance the creation date is not in the right (ISO-8601) format. 

For more (custom) validation we need the [validator](https://crates.io/crates/validator) crate.
```Cargo.toml
validator = { version = "0.15", features = ["derive"] }
```

Next we need to annotate the ```BlogEntry``` struct with our validation rules:
{{<highlight rust "linenos=table">}}
#[derive(Serialize, Deserialize, Clone, Debug, sqlx::FromRow, Validate)]
struct BlogEntry {
    created: DateTime<Utc>,
    #[validate(length(min = 10, max = 100, message = "Title length must be between 10 and 100"))]
    title: String,
    #[validate(email(message = "author must be a valid email address"))]
    author: String,
    #[validate(length(min = 10, message = "text length must be at least 10"))]
    text: String,
}
{{</highlight>}}

* derive Validate trait
* annotations on the properties

So far so good...

{{<highlight rust "linenos=table">}}
use axum::{http::StatusCode, Json, response::{IntoResponse, Response}, Router, routing::get, BoxError};
use axum::extract::{Extension, FromRequest, RequestParts, Json as ExtractJson};
use thiserror::Error;
use validator::Validate;
use async_trait::async_trait;

#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedJson<T>(pub T);

#[async_trait]
impl<T, B> FromRequest<B> for ValidatedJson<T>
    where
        T: DeserializeOwned + Validate,
        B: http_body::Body + Send,
        B::Data: Send,
        B::Error: Into<BoxError>,
{
    type Rejection = ServerError;

    async fn from_request(req: &mut RequestParts<B>) -> Result<Self, Self::Rejection> {
        let ExtractJson(value) = ExtractJson::<T>::from_request(req).await?;
        value.validate()?;
        Ok(ValidatedJson(value))
    }
}

#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] axum::extract::rejection::JsonRejection),
}

impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        match self {
            ServerError::ValidationError(_) => {
                let message = format!("Input validation error: [{:?}]", self).replace('\n', ", ");
                (StatusCode::BAD_REQUEST, message)
            }
            ServerError::AxumFormRejection(_) => (StatusCode::BAD_REQUEST, self.to_string()),
        }
            .into_response()
    }
}
{{</highlight>}}

Quite a bit of cruft! And we need more crates:
```Cargo.toml
thiserror = "1.0.29"
http-body = "0.4.3"
async-trait = "0.1"
```

The good news is that this code is generic for your whole application. So I guess you can put it in a separate file and largely forget about it.

So now If we now execute:
{{<highlight bash "linenos=table">}}
curl http://localhost:3000/entries -X POST -d '{"created":"2022-05-30T17:09:00.000000Z", "title":"aha", "author":"a", "text": "2"}' -v -H "Content-Type:application/json"
{{</highlight>}}

The server responds with a severe 400:BAD_REQUEST:

{{<highlight bash "linenos=table">}}
Input validation error: [ValidationError(ValidationErrors({
    "author": Field([ValidationError { code: "email", message: None, params: {"value": String("a")} }]), 
    "text": Field([ValidationError { code: "length", message: Some("text length must be at least 10"), params: {"value": String("2"),   "min": Number(10)} }]), 
    "title": Field([ValidationError { code: "length", message: Some("Title length must be between 10 and 100"), params: {"max": Number(100), "value": String("aha"), "min": Number(10)} }])}))]%       
{{</highlight>}}

And voila!

### Final remark
Check out the [crate documentation](https://docs.rs/validator/0.15.0/validator/) for all available checks. Like in javax.validation you can also define completely custom ones.