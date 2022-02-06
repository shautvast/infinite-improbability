---
title: "The rust option"
date: 2021-12-20T10:55:14+01:00
draft: true
---

`Option` is a well known construct in many languages. The first time I saw it was in scala, but soon after it appeared in java8. As java still means my livelyhood, it serves as my main frame of reference. I won't be covering all the details, because the java type is really a no brainer: it simply wraps a value _or not_ and what operations (methods on Optional) do depends on there being a value or not.
Rust Option starts with the same premisse. But immediately things start to diverge where it comes to actally using it. In rust the type is more or less mandatory for any optional value, because there is no (safe) `null` reference. So any struct or function that allows the absence of a value may use `std::option::Option` (or just `Option`) whereas in java it's (strangely) discouraged by static code analysis tools for class members and method arguments. Just use `null` they say, but I find it very useful to express the explicit possibility of null values as a valid state.

In a java application that uses IOC `null` is in many instances a 'not yet valid state' ie. a class instance has been constructed but its dependencies have not yet been injected. This is never a valid state once the application is fully constructed and so this would not be a case for an Optional value.

Rust's Option differs from java's Optional first and foremost in that in rust it is en enum. That would not be possible in java, because it's enums are limited in that an enum in java can only have attributes of the same type (for all enum values). Whereas in rust Option::None is different from Option::Some in that Some has a value ie Some("string") and None has not. 

In code:
```
pub enum Option<T> {
    None,
    Some(T),
}
```

construction:

|java.util.Optional<T>|std::option:Option<T>|
|---------------------|---------------------|
| of(T value)         | Some(value: T)      |
| ofNullable(T value) | Some(value: T)      |
* Some(null) will not compile.

