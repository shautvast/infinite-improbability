---
title: "The rust option"
date: 2021-12-20T10:55:14+01:00
draft: false
author: Sander Hautvast
---
![choose](/img/vladislav-babienko-KTpSVEcU0XU-unsplash.jpg)

`Option` is a well known construct in many languages. The first time I saw it was in scala, but soon after it appeared in java8. As java still means my livelyhood, it serves as my main frame of reference. I won't be covering all the details, because the java type is really a no brainer: it simply wraps a value _or nothing_ and what operations (methods on Optional) do depends on there being a value or not.

Rust Option starts with the same premisse. But immediately things start to diverge when it comes to actually using it. In rust the type is more or less mandatory for any optional value, because there is no (safe) `null` reference. So any struct or function that allows the absence of a value may use `std::option::Option` (or just `Option`) whereas in java it's (strangely) discouraged by static code analysis tools for class members and method arguments. Just use `null` they say, but I find it very useful to express the explicit possibility of null values as a valid state.

In a java application that uses _Dependency Injection_, `null` is in many instances a 'not yet valid state' ie. a class instance has been constructed but its dependencies have not yet been injected. This is never a valid state once the application is fully constructed and so this would not be a case for an Optional value. In java.

Rust's Option differs from java's Optional first and foremost in that in rust it is an _enum_. That would not be possible in java, because java enums are limited. In java enums can only have attributes of the same type (for all variants). Whereas in rust Option::None is different from Option::Some in that Some has a value ie Some("string") and None has not. 

In code:
{{<highlight rust>}}
pub enum Option<T> {
    None,
    Some(T),
}
{{</highlight>}}

Now you might say: _Option{value}_ is more expensive than '_a value or null_' because you have to wrap the value which costs more storage. For this rust has a trick up its sleeve, called _null pointer optimization_ that eliminates the extra memory use.
Read all about this (and more) [here](https://rust-unofficial.github.io/too-many-lists/first-layout.html).

construction:

|java.util.Optional<T>         |Option<T>|example: Option\<u32>|
|------------------------------|----------------|--------------|
| Optional.of(T value)         | Some(value: T) | Some(1)      |
| Optional.ofNullable(T value) | -              | -            |
| Optional.empty()             | None           | None         |


