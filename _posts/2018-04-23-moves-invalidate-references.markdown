---
layout: post
title:  "Moves Invalidate References"
date:   2018-04-23 08:00:00 -0500
categories: rust learning
---

Awhile back, I spent a couple days banging my head against a keyboard trying
to understand why Rust's borrow checker wouldn't accept a piece of code I had
written. The problem, perhaps not shockingly, involved references and
lifetimes. Specifically, it involved returning two structs from a function,
one of which held a reference to the other.

Understanding why my program was invalid required understanding a bit about
what a reference actually is in Rust. I have seen a number of people stumble
over this same problem in the last couple weeks, so I decided to explain how I
worked through the problem and what I learned in more detail.

## The Program

Let's start with a simple API that models the APIs where I have encountered the
problem before. This API allows us to create `Widget`s. Creating and using
`Widget`s requires some extra resources and work that we would like to abstract
from the client of our API, so we will encapsulate the resources and work
behind a `Context` type and require that every `Widget` reference a `Context`.
The naive approach gives us roughly the following:

{% highlight rust %}

struct Context { ... }

struct Widget<'c> {
  context: &'c Context,
  ...
}

impl<'c> Widget<'c> {
    pub fn new(context: &'c Context, ...) -> Self {
      ...
      Widget {
        context: context,
        ...
      }
    }
}

{% endhighlight %}

This approach is valid, although the presence of an explicit lifetime means
users of the API will also be forced to declare explicit lifetimes. Lets now
assume that a client consuming this API only wants to create one `Widget` and
so they try to encapsulate creating the `Context` and the `Widget` in a single
function like so:

{% highlight rust %}

fn create_context_and_widget() -> (Context, Widget) {
    let context = Context::new(...);
    let widget = Widget::new(&context, ...);
    (context, widget)
}

{% endhighlight %}

This seems like a fine thing to do. We create the `Context`, then the `Widget`,
and then return both. Since the `Context` is created before the `Widget` and
they are both returned (ie, not dropped) everything should live long enough and
the compiler should be happy. Right?

## The Problem

Wrong. First of all, if the client tries to compile the above function, they
will get an error complaining about a missing lifetime declaration.

```
error[E0106]: missing lifetime specifier
  --> src/main.rs:23:42
   |
23 | fn create_context_and_widget() -> (Context, Widget) {
   |                                             ^^^^^^^ expected lifetime parameter
   |
   = help: this function's return type contains a borrowed value with an elided
   lifetime, but the lifetime cannot be derived from the arguments
   = help: consider giving it an explicit bounded or 'static lifetime
```

A lot of the time when we are using references in Rust, the compiler can figure
out what the lifetimes on the references should be without requiring explicit
annotations. This is called [lifetime ellision][ellision]. In this case,
however, Rust cannot figure out what the lifetime of our `Widget` should be.

Let's just give the compiler what it's asking for, a lifetime parameter:

{% highlight rust %}

fn create_context_and_widget<'a>() -> (Context, Widget<'a>) {
  ...
}

{% endhighlight %}

Now the compiler gives us another error and we can start to see what the
problem is:

```
error[E0597]: `context` does not live long enough
  --> src/main.rs:25:31
   |
25 |     let widget = Widget::new(&context);
   |                               ^^^^^^^ borrowed value does not live long enough
26 |     (context, widget)
27 | }
   | - borrowed value only lives until here
   |
note: borrowed value must be valid for the lifetime 'a as defined on the function body at 23:1...
  --> src/main.rs:23:1
   |
23 | fn create_context_and_widget<'a>() -> (Context, Widget<'a>) {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

The compiler is complaining that the `Context` we created doesn't live long
enough. But we are returning it from the function.. it isn't being dropped..
what's the big deal?

The "gotcha" is that **returning owned values from a function causes them to be
moved** and that **moves invalidate references**.

In our `create_context_and_widget()` function, the `Context` we are creating in
the function is stack-allocated.[^1] It lives in one place in memory when it is
created and is moved to a new place in memory when the function returns. But
the reference we created in the function and passed to `Widget` points to the
original location and has no way of knowing that the data has moved to a new
location![^2]

## The Solutions

All of the solutions to this problem depend on making sure the `Context` is not
moved as long as there are `Widgets` that refer to it.

### The Boring Solution

The obvious, albeit unsatisfying, solution is to just not create both objects
in one function. This way, you can create the `Context` in some long-running
function that doesn't return until all `Widgets` that refer to it have been
dropped. This way, `Context` is never moved.

### The Less-Boring Solution

Another way to get around this solution is to allocate the `Context` on the
heap instead of on the stack. The motivation here is that if we allocate the
data on the heap, then it won't move until we release it and we should be able
to create references to it in the `create_context_and_widget` function that are
valid beyond the scope of the function. Here is a first attempt:

{% highlight rust %}

struct Context { ... }

struct Widget<'c> {
  context: &'c Box<Context>,
  ...
}

impl<'c> Widget<'c> {
    pub fn new(context: &'c Box<Context>, ...) -> Self {
      ...
      Widget {
        context: context,
        ...
      }
    }
}

fn create_context_and_widget<'a>() -> (Box<Context>, Widget<'a>) {
   let context = Box::new(Context::new(...));
   let widget = Widget::new(&context, ...);

   (context, widget)
}

{% endhighlight %}

Unfortunately, we still have the same problem here:

{% highlight shell %}

error[E0597]: `context` does not live long enough
  --> src/main.rs:25:30
   |
25 |    let widget = Widget::new(&context);
   |                              ^^^^^^^ borrowed value does not live long enough
...
28 | }
   | - borrowed value only lives until here
   |
note: borrowed value must be valid for the lifetime 'a as defined on the function body at 23:1...
  --> src/main.rs:23:1
   |
23 | fn create_context_and_widget<'a>() -> (Box<Context>, Widget<'a>) {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

{% endhighlight %}

The `Context` is being allocated on the heap now, but the `Box` we are using
to create it is still being stack-allocated and we have the same problem we had
before: when we return the `Box<Context>`, it is moved and the reference to it
becomes invalid. What we actually want here is an `Rc<Context>` instead of a
`Box`. `Rc` is like `Box` in that the data is heap allocated. However, it also
enables multiple owners of the data through reference counting (hence its
name). So instead of using the borrowed type `&Box<Context>` in `Widget`, we
can use the owned type `Rc<Context>` like so:

{% highlight rust %}

use std::rc::Rc;

struct Context {
  i: i32
}

impl Context {
   pub fn new() -> Self {
       Context { i: 0 }
   }
}

struct Widget {
   context: Rc<Context>,
}

impl Widget {
   pub fn new(context: Rc<Context>) -> Self {
       Widget { context }
   }
}

fn create_context_and_widget() -> (Rc<Context>, Widget) {
   let context = Rc::new(Context::new());
   let widget = Widget::new(Rc::clone(&context));

   (context, widget)
}

{% endhighlight %}

And this works! The `Rc<Context>` can be moved about freely because there are
no longer any references to it in our code. The `Rc` guarantees that the
`Context` will live as long as something refers to it and the location of the
heap-allocated `Context`, which is stored in a pointer in the `Rc`, never
changes.

## More about Rc

The implementation of `Rc` is essentially just a raw pointer and a reference
count that follows the following rules:

- When an `Rc` is cloned, the reference count goes up
- When an `Rc` is dropped, the reference count goes down
- When the reference count falls to 0, the memory is freed

Rust uses references and lifetimes to guarantee memory safety at zero cost.
The rules above also guarantee memory safety in a different way then references
and lifetimes do, but they incur an additional cost: the data must be heap
allocated and the reference count must be tracked. This is a common theme in
Rust: provide zero-cost, safe abstractions by default, but enable programmers
to opt-in to more costly abstractions as needed. For more information on `Rc`,
[read the docs][rc].

[ellision]: https://doc.rust-lang.org/nomicon/lifetime-elision.html

[rc]: https://doc.rust-lang.org/std/rc/

[^1]: When a value is created in a function, the data used to represent the
      value is allocated on the stack. When a function returns, all the data
      allocated by the function call on the stack is deallocated or "popped off
      the stack". If a function returns some value to the calling function, it
      is _moved_ from its location in the stack in the called function to a new
      location in the calling function instead of being deallocated.

[^2]: References are basically just fancy pointers. They point to some address
      in memory and Rust guarantees the pointer is always valid. So if we
      create a reference to some data, Rust guarantees that the data does not
      move as long as there is a reference to it. If this cannot be guaranteed,
      the compiler will not accept your program.
