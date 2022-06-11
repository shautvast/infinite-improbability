---
title: "Get Enterprisey with Rust part 4 - Constants and regular expressions"
date: 2022-06-11T11:27:41+02:00
draft: false
---

This is the fourth part of the series on everyday programming tasks in the average CRUD application.

So far we covered:
1. [initial setup with axum and postgres, logging and dates](/enterprisey)
2. [input validation on incoming Json objects](/enterprisey2)
3. [working with environment variables](/enterprisey3)

This time we're going to look at another humble task: regular expressions and constants (often seen together).

`!warning!`
As it turns out this post dives deeper into some rust intricacies than the previous. So be cautious! You might learn something...

The beginning is not going to be complicated. We need two new crates:
```Cargo.toml
regex = "1.5"
lazy_static = "1.4"
```

First you will see how regular expressions work and next, how to make sure they are only compiled once (using constants), which is what you want, for performance.

### Raw strings

Imagine you need to remove punctuation from a sentence. For this you could use this regular expression:
```
[\.:,"'\(\)\[\]|/?!;]+
```

The [regex](https://crates.io/crates/regex) crate uses perl style expressions which is also what java does.

To make this work in rust:
{{<highlight foo>}}
use regex::Regex;

let punct_re = Regex::new(r#"[\d\.:,"'\(\)\[\]|/?!;]+"#).unwrap();
{{</highlight>}}

I did not use highlighting in this snippet, because the highlighter on this page is actually incorrect in that it doesn't 'escape' the double-quote character in the middle, thinking it's the end of the string. The code uses a _raw string_:

`r#"..."#`

If I hadn't included a double-quote as part of the expression, this would have been valid as well: 

`r"..."`

 And if I wanted to include a pound-sign (#) in the expression, I would need to write this:

`r##"..."##`

This syntax avoids having to use escaping with backslash and makes the expression more readable. 

The newly created ```punct_re``` expression can simply be used like this:

{{<highlight rust>}}
let it_contains_punctuation = punct_re.is_match("!");
{{</highlight>}}

Check [the docs](https://docs.rs/regex/latest/regex/) for more information on all available methods.


### Replace all

In our case we need to use ```replace_all``` and pass an empty string to effectively remove all unwanted characters:
{{<highlight rust>}}
let result = punct_re.replace_all("hello world!", "");
{{</highlight>}}

Now it will get a little bit tricky, because `replace_all` does not return a String or string slice, but instead a Cow...

![meuh](/img/patrick-baum-hB9vo06o9z8-unsplash.jpg)
_No not you!_

A COW as in Clone On Write:
```
A clone-on-write smart pointer.

The type Cow is a smart pointer providing clone-on-write functionality: it can enclose and provide immutable access to borrowed data, and clone the data lazily when mutation or ownership is required. The type is designed to work with general borrowed data via the Borrow trait.

Cow implements Deref, which means that you can call non-mutating methods directly on the data it encloses. If mutation is desired, to_mut will obtain a mutable reference to an owned value, cloning if necessary.
```

_What is this and why is it used in `replace_all`?_

To start with the latter, it was put in for efficiency, returning a reference to the original string in case nothing needed replacing. And a `Cow` allows mutation, as opposed to other smart pointers (like `Box` or `Rc`), which is useful when you do need to replace. 

If you want you can read more [here](https://github.com/rust-lang/regex/issues/676)

As the docs state:
`Cow` implements `Deref`

Which means that something like the C-language `*` operation for pointers is automatically applied by the compiler to turn the smart pointer to a value, into the value itself.

{{<highlight rust "linenos=table">}}
use std::borrow::Cow;

let result: Cow<str> = punct_re.replace_all("hello world!", "");
let result: &str = &result;
{{</highlight>}}

I have included the types to show what goes on and because line 4 wouldn't compile without it. 
1. you get the result as `Cow<str>`
2. you say you want a string slice, so the compiler deref's the `Cow` to `str`. 
3. and you get a new reference `&` to `result`.

Without dereferencing you would get a `&Cow<str>` instead, which isn't helpful at all.

One last thing: `let result` twice? Yes, that's rust's [shadowing](https://doc.rust-lang.org/stable/rust-by-example/variable_bindings/scope.html). Really handy to avoid (quasi) hungarian notation.

### constants

Rust has a `const` keyword:

{{<highlight rust>}}
const A: usize = 1;
{{</highlight>}}

But 
{{<highlight rust>}}
const punct_re: Regex = Regex::new(r#"[\d\.:,'\(\)\[\]|/?!;]+"#).unwrap();
{{</highlight>}}
is not allowed! Because it contains a function call, so the actual value cannot be determined until after compilation.

To work around this we need [lazy_static](https://crates.io/crates/lazy_static).

This is a `macro` and the code that you put in it is guaranteed to only run once. 

We could simply put it in a function, right where we need it:
{{<highlight rust "linenos=table">}}
pub fn clean(text: &str) -> String {
    lazy_static! {
        static ref PUNCT: Regex = Regex::new(r#"[\d\.:,"'\(\)\[\]|/?!;]+"#).unwrap();

    }

    String::from(PUNCT.replace_all(text, ""))
}
{{</highlight>}}
(_note that I took the `"` out, and used highlighting again_)

`!important!` I cannot use `&str` here, because returning a reference from a function is in fact a _dangling pointer_. That is a pointer to memory that is owned by the function and reclaimed when it finishes. That's why we have to copy the value to an owned `String` and return that. This has a performance impact. Try to avoid copying as much as possible!

**Conclusion**

Working with regular expressions and constants isn't really difficult, but it opens the door to some more advanced concepts in the rust type system. 

I highly recommend https://rust-unofficial.github.io/too-many-lists/index.html. Don't just read it. Don't copy-paste the code. Don't even copy it manually.

_Read it, hide the browser tab, and try to create the code of a variation on the linkedlist by heart._ Reopen the tab whenever you are stuck. And don't despair!

