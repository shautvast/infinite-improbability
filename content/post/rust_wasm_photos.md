---
title: "Let's create an app in webassembly"
date: 2022-02-05T20:11:08+01:00
draft: false
author: Sander Hautvast
---
![soup assembly](/img/markus-winkler-08aic3qPcag-unsplash.jpg)
Web assembly is cool and new and it lacks nice how-to's for tasks that are by now quite mundane for most web developers. So let's get started using [yew](https://yew.rs/). 

What is yew? 

'Yew is a modern Rust framework for creating multi-threaded front-end web apps using WebAssembly.' (yew.rs)

I actually tried several other rust/wasm frameworks, but this is the first that didn't end in tears and compile errors. The documentation is quite good. There's lots of examples. And it works well in _stable_ rust!

I will not explain rust specific stuff. There are other tutorials already. Instead I want to focus on creating a simple webapp. It will have a 'drop zone' for dragging and dropping images from your local harddrive that will then be put on the screen (no upload to a server). This is really simple, but still requires a lot more code than a simple 'hello world' and will introduce the rust bindings for web api's that you probably already know.  

**Install yew**

You can read all about how to install it [here](https://yew.rs/docs/tutorial).
In short:
* install rust if you haven't already
* install trunk: ```cargo install trunk```
[Trunk](https://trunkrs.dev/) is like webpack for wasm
* ```rustup target add wasm32-unknown-unknown``` 


**Create a project**

```cargo new yew-app```
This creates a regular rust project with a Cargo.toml and a main.rs.

**Add a skeleton index.html**

In the project root, create ```index.html``` and enter:
{{<highlight html "linenos=table">}}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>dropping images using wasm and yew</title>
    <style>
    html {
      font-family: 'Gill Sans', 'Gill Sans MT', Calibri, 'Trebuchet MS', sans-serif;
      color: rgb(20, 20, 104);
    }

    .drop-zone {
      border: 4px dashed rgb(17, 122, 184);
      width: 200px;
      height: 100px;
      background-color: rgb(202, 202, 238);
    }

    p {
      display: table;
      margin: 0 auto;
      transform: translateY(50%);
    }
  </style>
  </head>
  <body>
  </body>
</html>
{{</highlight>}}

**Start serving**

```trunk serve``` will start a webserver on [http://localhost:8080](http://localhost:8080). 

Bonus: it will pick up _any_ changes and refresh your website accordingly. I wouldn't wanna have it any other way!

**Add boilerplate**
* Add these dependencies to ```Cargo.toml```
{{<highlight toml "linenos=table">}}
gloo-utils = "0.1.2"
log = "0.4.6"
wasm-bindgen = "0.2.79"
wasm-logger = "0.2.0"
web-sys = {version = "0.3.56", features = [
  "DataTransfer",
  "DataTransferItemList",
  "DataTransferItem",
  "Document",
  "Element",
  "HtmlImageElement",
  "Url",
  'Blob'
]}
yew = "0.19"
{{</highlight>}}

* In src add a new file: ```app.rs```
* Make src/main.rs look like:
{{<highlight rust "linenos=table">}}
mod app;

fn main() {
    wasm_logger::init(wasm_logger::Config::default());
    yew::start_app::<crate::app::DropPhoto>();
}
{{</highlight>}}

This is actually not much too much in terms of boilerplate. There is a lot of power included in these crates!

**Add a component**

BTW I mostly use visual studio code for writing rust. Intellij does the job equally well. Neovim with the appropriate plugins (as always) is also an option, but I'm not an nvim wizard and I haven't got it to work quite as well as for example [Jon Gjengset](https://thesquareplanet.com/) (check out his [youtube channel](https://www.youtube.com/c/JonGjengset)).

We'll be adding all code to  ```app.rs```, so open that file in your IDE.

**Tip** 

Avoid copy-paste! I found that simply copying code from blogs like this leaves you no smarter than you are right now. Actually typing in the code by hand will improve retention of what you actually did. 

**Added bonus** 

Any typing mistakes will potentially leave you in the dark because the compile error you get is unclear to you. This will make you look harder for differences and you may even end up debugging and whatnot, which will help you remember even better!

{{<highlight rust "linenos=table">}}
use gloo_utils::document;
use wasm_bindgen::JsCast;
use web_sys::{Url};
use web_sys::{DragEvent, HtmlImageElement};
use yew::{html, Component, Context, Html};

enum Msg{
    Dropped(DragEvent),
    Dragged(DragEvent),
}

struct DropImage{
    images: Vec<String>,
}

impl Component for DropImage {
    type Message = Msg;
    type Properties = ();

    fn create(_ctx: &Context<Self>) -> Self {
        Self { images: vec!()}
    }

    fn update(&mut self, _ctx: &Context<Self>, msg: Self::Message) -> bool {
        match msg {
            Msg::Dragged(event) => true,
            Msg::Dropped(event) => true,
        }
    }

    fn view(&self, ctx: &Context<Self>) -> Html {
        let link = ctx.link();
        html! {
            <>
            <div class="drop-zone" 
                ondragover={link.callback(|e| Msg::Dragged(e))} 
                ondrop={link.callback(|e| Msg::Dropped(e))}>
                <p>{ "drag your images here" }</p>
            </div>
            <div id="images"></div>
            <div>{ self.images.iter().collect::<Html>() }</div>
            </>
        }
    }
}
{{</highlight>}}

* you added an enum
* you added a struct
* you implemented the trait yew::Component for the DropImage struct

So DropImage is now a _Component_, much the same way as in for instance _Angular_ or other frameworks. The component is the object that will maintain your state, update the view and respond to events. A component has at least two methods: ```create``` and ```view```. Often it will also include ```update```.

```create``` must return the struct so this is the place to add initial state values, here an empty list of the names of the files that will be dragged in. The injected &Context reference can be used to register callbacks.

```view``` determines how the component is rendered to the DOM. This looks like jsx in _React_. Use the html! macro to create html. This is probably more convenient than to do it programmatically, but that is also an option. 

There are some differences with regular html to be aware of. All _text_ must be surrounded by curly braces {}. A constant string in quotes will simply be turned into the text html child, but you can output any component value, eg: ```{self.value}```

Note that two event handlers, ```ondragover``` and ```ondrop``` are registered in the drop-zone div. What does ```{link.callback(|e| Msg::Dragged(e))}``` mean? It sends a message called Msg::Dragged with a payload that is the raised html event (e). The component is now be able the handle this message. For this you need:

```update``` is called by the framework and it receives an instance of the Msg enum and it will respond by choosing appropriate action. This could mean update the internal component state or the view directly. I fact I doubt if the latter is really what you would want. In fact we could have defined the _images_ div as follows
{{<highlight html>}}
<div id="images">{ images.iter().collect::<Html>() }</div>
{{</highlight>}}
In general you should let the view reflect component state, instead of hacking the DOM. But this doesn't work here because there is no formatter for image objects, so we will add them to the DOM in the update method itself. (I'm open for anything better). Instead I added another div, which does iterate over the image names, which are strings. This implementation is utterly naive and ugly (for the end user). The aim is simply to show two ways to update the view.

**Add Event handling**

We will now be updating the ```update``` method as follows:
{{<highlight rust "linenos=table">}}
match msg {
    Msg::Dragged(event) => {
        event.prevent_default();
        false
    }
    Msg::Dropped(event) => {
        event.prevent_default();
    }
}
{{</highlight>}}

The first step is preventing default browser behaviour for dragging and dropping. This is the same as what would do in javascript or typescript. All the usual methods (here ```preventDefault```) are available but in snake case as is the way of rust. 

(It takes some code reading to find the right methods. I think we need more documentation than mere references to MDN.)

Now we just have to add the following after prevent_default in the Msg::Dropped case (between lines 7 and 8 in the above code).

{{<highlight rust "linenos=table">}}
// access DataTransfer api
let data_transfer = event.data_transfer().expect("Event should have DataTransfer");
let item_list = data_transfer.items();
for i in 0..item_list.length() {
    let item = item_list.get(i).expect("Should find an item");
    if item.kind() == "file" {
        let file = item.get_as_file().expect("Should find a file here").unwrap();        
        
        // create img element
        let element = document().create_element("img").unwrap();
        let img = element.dyn_ref::<HtmlImageElement>().expect("Cannot create image element");
        let url = Url::create_object_url_with_blob(&file).expect("Cannot creat url");
        img.set_src(&url);
        img.set_width(100);
        img.set_height(100);

        // append it to container div
        if let Some(images) = document().get_element_by_id("images") {
            images.append_child(img).expect("Cannot add photo");
        }

        // update component state
        self.images.push(file.name());
    }
}
true
{{</highlight>}}

This is quite a bit of things going on. But it was more or less taken directly from the javascript on MDN [here](https://developer.mozilla.org/en-US/docs/Web/API/File/Using_files_from_web_applications) and [here](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API). 

Doesn't this look familiar?
```rust
document().create_element("img")
```

Make sure to have all the imports (```use``` statements)  right, or this will not compile and it will not be obvious why. The ```true``` return value means the engine has to rerender the view after an update of the component. 

**Final thoughts**
* I only tested it in firefox, no guarantees.
* OMG, I changed my code while writing this post. A definite no-no. I hope the code works. Otherwise check the [repo](https://github.com/shautvast/dragndrop_wasm_yew) ...sorry.
* Actually that is kinda interesting: while explaining the code, I thought, "well that (code) doesn't really make sense", and I found a better solution.
* Yew allows for other ways to handle events for instance. I ended up with what I found most elegant.
* There's more to yew. Read the docs!
* I used inline css here, but you don't have to. 
* There's undoubtedly room for more improvements. Hey, I'm still learning!
* The workflow/structure is pretty solid: good old MVC pattern.
* The code is definitely more 'technical' than what regular webdevs write nowadays. Is this to be mainstream stuff in the near future? Or will it occupy a high performance niche? I am guessing that for most web developers more abstraction is needed, in framework or language support... 
* But on the whole I think WASM and Yew are up to it: redefine web apps once again!


