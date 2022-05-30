---
title: "Rust for Java developers, Introduction"
date: 2021-12-17T13:07:49+01:00
draft: false
author: Sander Hautvast
---
![rusty bridge](/img/christopher-burns--mUBrTfsu0A-unsplash.jpg)

**Manifesto**

ahum..

It is my conviction that rust should should rule the world! Not only because it is cool new technology, but first and foremost because it requires less computing resources (cpu cycles and memory) which also translates to natural resources (electricity). Since large parts of the grid are still powered by fossile, rust in the end is a way to cut back on CO2 emissions! _Especially_ compared to java which is still resource hungry, despites lots and lots of improvements over the years. 

And java powers much of the web. Rust, while being regarded as a _systems_ language is a perfectly feasible choice for any web application. Yet it's uptake is slow, in part because of the learning curve. This blog is intended to help developers make the change. There is already splendid documentation out there. I hope that I can add some information that I have found useful, once I got my head around it.

**tl;dr**
Do not think 'Rust is just a new language, I can map what I already know onto some new keywords'. Rust is different. But taking the effort will pay off.

In this post you will see some syntax and get a feel for how rust differs from other languages because of its implementation of _ownership_.

Imagine a language in which duplicating a valid line of code slaps you in the face with a compilation error:
{{<highlight rust "linenos=table">}}
fn main() {
    let name = String::from("Bob");
    let msg = get_hello(name);
    println!("{}", msg);

    // let msg2 = get_hello(name); -- ERROR Use of moved value
    // println!("{}", msg2);
}

fn get_hello(name: String) -> String{
    let mut text = "Hello ".to_owned();
    text.push_str(&name);
    return text;
}
{{</highlight>}}

__line 6 is a copy of line 3. The first compiles but the second does not. WTF?__

So what is going on? Let's look at it line by line.

Starting at line 1 it seems easy enough, and it is: `fn` defines a function, in this case `main` which is the entrypoint for any rust application. We could add `pub` to make it public, but in this case that wouldn't be necessary. 
Next `()`, like in a lot of languages means: put any parameters here. Curly braces: we have block scope.

`let` defines a variable, in ths case `name`. Some things to say about this:
For one: rust is _strongly typed_, so we could also say `let name: String = String::from("...")` but rust infers the type as much as possible (and that is much more than let's say java does). 

Actually `let` does more than declare and assign, it establishes ownership. More on that later. 

One last thing to mention here: 
this is allowed:
{{<highlight rust>}}
let name;
name = String::from("<your name here>");
let msg = get_hello(name);
{{</highlight>}}
and this is not:
{{<highlight rust>}}
let name;
// let msg = get_hello(name); -- ERROR Use of possibly uninitialized variable 
{{</highlight>}}

because no value is actually assigned to `name`. In java we could cheat and assign `null` to name, but rust won't have any of that. 

There is no `null` in rust. 

This is when you say: _'Thank god for that!'_

To which I would reply: _'Welll... actually there is'_, but that is an advanced topic, and beyond the scope of this post. Let's just stick to the good habit of never having null-references and life is fine.

_So much going on in just 2 lines of code...._

About strings: There is String and there is &str (pronounced string-slice). Javans are allowed to apply the following mapping in their head:
| Rust |Java         |
|------|-------------|
|String|StringBuilder|
|&str  |String       |

So rust String is mutable and &str is not. The latter is also a (shared) reference. This is important. It means I have **borrowed** something, I can use it, but I am not allowed to change it and I am certainly not the **owner**. More on ownership later...

On to the next two lines. 
{{<highlight rust "linenos=table,linenostart=3">}}
let msg = get_hello(name);
println!("{}", msg);
{{</highlight>}}

Nothing too fancy: we call a function and print the result. Rust has easy string interpolation using `{}` for a value that you want to insert into a string. The print! statement ends with an exclamation mark and this signifies that you're dealing with a *macro*. So it compiles to something completely different, but who cares about that?

Next: 
{{<highlight rust "linenos=table,linenostart=10">}}
fn get_hello(name: String) -> String {
{{</highlight>}}
This function actually has a return value which unlike java and C is at the end of the declaration (following the arrow```->```). I find this more intuitive.

{{<highlight rust "linenos=table,linenostart=11">}}
let mut text = "Hello ".to_owned();
{{</highlight>}}

Here we create a String, concatenate the input argument and return the result. Make sure the test variable is `mut`. The method `to_owned` creates a copy of a str slice in the form of an owned String.

The `mut` keyword indicates that we are actually allowed to change the the value of `some_string`. I mentioned earlier that String is mutable, but if you actually want to mutate it, you have to add `mut`. And: **values aren't mutable (or not), bindings are!**

So:
{{<highlight rust>}}
let x = String::from("x");
//x.push_str("NO!"); // compilation error, x is not mutable
let mut y = x;
y.push_str("YES!"); // but this compiles!
{{</highlight>}}

Weird, right? Well remember that by default nothing is mutable. And that things can magically 'become' mutable through `mut`. 

But be aware: What do you think would happen if you'd add 
`get_hello(x);`?
It would not compile. Ownership has passed from `x` to `y`. Now, sit back and just let this sink into your mind...

Still here? After ```y=x```, ```x``` can no longer be accessed!

So in this case, there is never a situation in which a value is mutable and immutable at the same time. The compiler won't have it. This way it prevents a **lot** of bugs. The rust compiler is quite famous for handing you friendly reminders that your code has issues, even suggesting the fix...

Next line:
{{<highlight rust "linenos=table,linenostart=12">}}
    text.push_str(&name);
{{</highlight>}}

What's interesting here is that we pass a reference to `name`, using the ampersand `&`. A reference to a String is by definition a string-slice. So the signature for push_str is:

{{<highlight rust>}}
pub fn push_str(&mut self, string: &str)`
{{</highlight>}}

Oh and this is interesting: the `self` keyword is sort of like `this` in java, but not quite. See `this` is the reference to the object you act on in java. In rust, like python, if we have a function on an object (also called a method), the first argument is always `self`. 

There are objects in rust, but they're called `struct`, and they are similar in some respects, but there's no inheritance. Remember: _favour composition over inheritance_.

Okay, one more line to go:
{{<highlight rust "linenos=table,linenostart=13">}}
return text;
{{</highlight>}}

You might have noticed that this could have been java code, except that rust prefers _snake_case_ for local variables. The compiler actually enforces this. You can turn it off, but that would be rude. 
The funny thing about this line is that in practice it will always be written like this:

{{<highlight rust "linenos=table,linenostart=13">}}
text
{{</highlight>}}

`return` is optional (handy for early returns), but it needs the semicolon. It's shorter without them, so why not?

Be aware that is more powerful than it seems at first glance. The last line(s) of anything, provided that is does not end with `;` is always evaluated as an expression and returned to the caller. So it could have been a compound `match` expression (fetaured in a later blog) or a simple `if` (which is also an expression!):

{{<highlight rust>}}
fn foo(bar: bool) -> u32{
    if bar {
        1
    } else {
        0
    }
}
{{</highlight>}}

This is so much cleaner than anything else!

__ownership__

I saved the best for last here, but I have already given some hints as to what this is. Please memorize the following rules:
1. There is one, and only one _owner_ of a value
2. Ownership is scoped by blocks: at the end of the containing block, a value is dropped.
3. I may lend _shared references_ to any number of any number of borrowers.
4. I may at any given time lend only _one_ _unique_ reference to a borrower. I remain owner.
5. I may pass on ownership, but then I can no longer use it in any way.
6. Reassigning a value to a new binding means transfer of ownership.
7. A function parameter becomes owner when the function is  called. (unless it's a reference)

The latter is the reason that I could not call `get_hello` *twice* with `name` as the argument. At the first call, my ownership is passed on. The function, being the owner, can do with as it sees fit, but at the end of the function, the the value is dropped and you can no longer use it. Once you have updated your mental models with this knowledge, your life as a rust developer has become a little easier.

Astute readers may have noticed the `&mut` before `self` in the function declaration for push_str. This is what a unique reference looks like. So you get a mutable reference to self, which means that you can mutate `self` (here append some text to the string), but you have not become the owner. The mutable reference to self goes out out scope once the method ends and the original owner can carry on using it. This is true for any `&mut`.

If you have made it this far, I recommend you start reading [the book](https://doc.rust-lang.org/book/)
and install [rustup](https://www.rust-lang.org/tools/install). Enjoy!
