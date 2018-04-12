---
layout: post
title:  "Are Rust's Borrow Rules Only Helpful in Multithreaded Programs?"
date:   2018-04-12 21:08:00 -0500
categories: rust learning
---

Rust's [borrow rules][borrow-rules] are one of its most distinctive qualities.
They are pretty straightforward:

> At any given time, you can have either but not both of:
> * One mutable reference.
> * Any number of immutable references.

What is less straightforward is _why_ Rust has these borrow rules, so I decided
to find out. That you can only have one mutable reference at a time is
especially puzzling when you consider this is something you do all the time in
other languages:

{% highlight python %}
x = [1, 2, 3]
y = x
x.append(1)
y.append(2)
assert(x == y)
{% endhighlight %}

Let's be good programmers and RTFD. [The Rust Programming
Language][data-race-answer] has the following to say on the borrow rules:

> This restriction allows for mutation but in a very controlled fashion...most
> languages let you mutate whenever you’d like. The benefit of having this
> restriction is that Rust can prevent data races at compile time.

This seems to answer the question, but I found it pretty unsatisfying. Rust is
one of my favorite languages because so many of its design decisions have been
well thought out and have had the pros and cons weighed carefully. If you read
through the project's code of conduct, you will find this particularly
insightful observation:

> [E]very design or implementation choice carries a trade-off and numerous
> costs. There is seldom a right answer.

Placing borrow rules' restrictions on the entire language in order to serve
what is only a problem in multithreaded programs seemed a bit... unbalanced. It
wasn't until I started reading the [Rustonomicon][nom-answer] that I found the
answer. Buried deep in that tomb of dark secrets (okay it is like the third
page) is the real reason why you can only have one mutable reference at a time.
Here is the example used to demonstrate the point:

{% highlight rust %}
let mut data = vec![1, 2, 3];
// get an internal reference
let x = &data[0];

// OH NO! `push` causes the backing storage of `data` to be reallocated.
// Dangling pointer! Use after free! Alas!
// (this does not compile in Rust)
data.push(4);
{% endhighlight %}

The above does not compile because `Vec::push()` takes `&mut self` as its
receiver, which would require a mutable and an immutable reference to `data` to
exist at the same time, which would be in violation of the borrow rules.

The comment clarifies why this is bad. When you push onto a `Vec`, there is a
chance that it will grow, which means a new chunk of memory will be requested
and the data will be relocated to that new chunk. If this were to happen while
you had, say, another reference to that data, you would need to update all the
references to point to the new data or watch your program do strange things
(or just crash) because your pointers are dangling. 🤯

If you read on, there is also a section on pointer aliasing and how the borrow
rules prevent client code from doing silly things like passing pointer
arguments that overlap in memory.

At this point, if you are like me, you probably want to ask **BUT WHAT IF I
KNOW WHAT I AM DOING AND IT ISN'T BAD CAN YOU PLEASE JUST LET ME DO IT**?

It turns out you can...sort of. Rust's standard library provides the
[`RefCell`][refcell] type which enables the "interior mutability" pattern.
Interior mutability lets you push enforcement of the borrow rules into runtime.

Rust's ownership model requires your object dependency graphs to be trees and
that every node in the tree is either mutable or immutable. Since you cannot
create a mutable reference from an immutable reference, you have trees where
there is a point at which references change from mutable to immutable and then
stay that way.

        A   &mut
       / \
     & B |
       | C  &mut
       |\
     & D |
         E  &mut <- you can't do this..

By pushing enforcement of the borrow rules to runtime, `RefCell` let's you
switch back to mutable references in a controlled way:

        A   &mut
       / \
     & B |
       | C  &mut
       |\
     & D |
         E  RecCell<T> <- you can turn this into an &mut at runtime

# Further Reading

For a worked example of how Rust's borrow rules solve many common memory safety
problems in other systems programming languages, see [Memory Safety in Rust: A
Case Study with C][case-study].

[borrow-rules]: https://doc.rust-lang.org/book/second-edition/ch04-02-references-and-borrowing.html#mutable-references
[data-race-answer]: https://doc.rust-lang.org/book/second-edition/ch04-02-references-and-borrowing.html#the-rules-of-references
[nom-answer]: https://doc.rust-lang.org/nomicon/ownership.html
[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[case-study]: http://willcrichton.net/notes/rust-memory-safety/
