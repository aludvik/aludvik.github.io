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
over this same problem in the last couple weeks, so I thought I'd work through
where the problem is often encountered, why the program is invalid, and what
the possible solutions are.

## The Program

We are going to start by describing a somewhat abstract API that allows us to
create `Widget`s. Creating and using `Widget`s requires some extra resources
and work that we would like to abstract from the client of our API, so we will
encapsulate the resources and work behind a `Context` type and require that
every `Widget` references a `Context`. The naive approach gives us roughly the
following:

{% highlight rust %}

struct Context { ... }

struct Widget<'c> {
  context: &'c Context,
  ...
}

impl<'c> Widget<'c>' {
    pub fn new(context: &'c Context, ...) -> Self {
      ...
      Widget {
        context: context,
        ...
      }
    }
}

{% endhighlight %}

There is nothing wrong with the above approach, although we are declaring an
explicit lifetime parameter. (There's nothing wrong with having explicit
lifetime parameters in structs, but it is going to cause us problems in a
second.) Sometimes a client consuming the library will only want one `Widget`.
The problem occurs when they try to encapsulate creating the `Context` and the
`Widget` in a single function:

{% highlight rust %}

fn create_context_and_widget() -> (Context, Widget) {
    let context = Context::new(...);
    let widget = Widget::new(&context, ...);
    (context, widget)
}

{% endhighlight %}

This seems like a fine thing to do. Presumably we will create the `Context`,
then the `Widget`, and then return both. Since the `Context` is created before
the `Widget` and they are both returned (ie, not dropped) everything should
live long enough and the compiler should be happy. Right?

## The Problem

Wrong. First of all, if you try to compile the above function, you will get an
error complaining about a missing lifetime declaration.

{% highlight shell %}

error[E0106]: missing lifetime specifier
  --> src/main.rs:23:42
   |
23 | fn create_context_and_widget() -> (Context, Widget) {
   |                                             ^^^^^^^ expected lifetime parameter
   |
   = help: this function's return type contains a borrowed value with an elided
   lifetime, but the lifetime cannot be derived from the arguments
   = help: consider giving it an explicit bounded or 'static lifetime

{% endhighlight %}

A lot of the time when we are using references in Rust, the compiler can figure
out what the lifetimes on the references should be without requiring explicit
annotations (this is called [lifetime ellision][ellision]. In this case,
however, Rust cannot figure out what the lifetime of our `Widget` should be.

Let's be naive some more and just give the compiler what it's asking for: a
lifetime parameter:

{% highlight rust %}

fn create_context_and_widget<'a>() -> (Context, Widget<'a>)

{% endhighlight %}

Now the compiler gives us another error and we can start to see what the
problem is:

{% highlight shell %}

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

{% endhighlight %}

The compiler is complaining that the context we are creating doesn't live long
enough. But we are returning it from the function.. It isn't being dropped..
what's the big deal? The "gotcha" is that **moves invalidate references**.

Roughly speaking, when a value is created in a function, the data used to
represent the value is allocated on the stack. When a function returns, all the
data allocated by the function call on the stack is deallocated. If a function
returns some value to the calling function, it is _moved_ from its location in
the stack in the called function to a new location in the calling function
instead of being deallocated.

In Rust, references are basically just fancy pointers. They point to some
address in memory and Rust guarantees the pointer is always valid. So if we
create a reference to some data, Rust guarantees that the data lives at least
as long as the reference (this is what it uses lifetimes for).

Back to our `create_context_and_widget()` function, the problem is that the
`Context` we are creating in the function is located at one place in memory
when it is created and is moved to a new place in memory when the function
returns. But the reference we created in the function and passed to `Widget`
points to the original location and has no way of knowing that the data has
moved to a new location!

## The Solutions

All of the solutions to this problem depend on making sure the `Context` is not
moved as long as there are `Widgets` that refer to it.

### The Boring Solution

The obvious, albeit unsatisfying, solution is to just not create both objects
in one function. This way, you can create the `Context` in some long-running
function that doesn't return until all `Widgets` that refer to it have been
dropped.

### The Less-Boring Solution

Another way to get around this solution is to allocate the `Context` on the
heap instead of on the stack. The motivation here is that if we allocate the
data on the heap, then it won't move until we release it and we should be able
to create references to it in a function that are valid beyond the scope of the
function. Here is a first attempt:

{% highlight rust %}

struct Context { ... }

struct Widget<'c> {
  context: &'c Box<Context>,
  ...
}

impl<'c> Widget<'c>' {
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
to create it actually stack allocated and we have the same problem we had
before: when we return the `Box<Context>`, it is moved and the reference to it
becomes invalid. What we actually want here is an `Rc<Context>` instead of a
`Box`. `Rc` is like `Box` in that the data is heap allocated. However, it
also enables multiple owners of the data through reference counting (hence its
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

And this works! We no longer have any explicit lifetimes in our code so we
don't need to worry about things moving and the `Rc` guarantees that the
`Context` will live as long as there is a type that refers to it.

[ellision]: https://doc.rust-lang.org/nomicon/lifetime-elision.html
