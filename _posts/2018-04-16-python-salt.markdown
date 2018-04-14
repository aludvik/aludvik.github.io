---
layout: post
title:  "TIL - Python has salty hashes"
date:   2018-04-16 08:00:00 -0500
categories: python cryptography
---

TIL - Python uses a salt when hashing by default. This means that when you
iterate over things like dicts and sets, which depend on hashing to store data
efficiently, you can get a different iteration order across different
instantiations of the same program. However, iteration order seems consistent
within the same program (presumably because the salt is determined when the
process is created).

So for example, when I run the following, it never raises an exception:

{% highlight python %}

# salty_hashes.py

d = {
  'alice', 'bob', 'charlie', 'danika', 'earl', 'florence', 'george',
  'harriet', 'isabel', 'john',
}

order = [k for k in d]
assert(all(order[i] for i, k in enumerate(d)))

{% endhighlight %}

However, the following will non-deterministically raise an exception if run
repeatedly:

{% highlight python %}

import os

d = {
  'alice', 'bob', 'charlie', 'danika', 'earl', 'florence', 'george',
  'harriet', 'isabel', 'john',
}

order_file = "order"
if os.path.isfile(order_file):
  order = open(order_file).read().split(" ")
else:
  order = [k for k in d]
  open(order_file, "w").write(" ".join(order))

assert(all(k == order[i] for i, k in enumerate(d)))

{% endhighlight %}

{% highlight bash %}

$ python3 salty_hashes.py
$ python3 salty_hashes.py
Traceback (most recent call last):
  File "salty_hashes.py", line 15, in <module>
    assert(all(k == order[i] for i, k in enumerate(d)))
AssertionError

{% endhighlight %}

This was different from my mental model, which was that iteration over
dictionaries was unpredictable and shouldn't be relied on, but consistent
across runs.

The salt used with hashing can be controlled by setting the PYTHONHASHSEED
environment variable. To make runs consistent across processes, just set the
it before you run your command:

{% highlight bash %}

$ PYTHONHASHSEED=1 python3 salty_hashes.py
$ PYTHONHASHSEED=1 python3 salty_hashes.py
$ PYTHONHASHSEED=1 python3 salty_hashes.py

{% endhighlight %}

# Further reading:

You can read more about how Python does hashing and the security
vulnerabilities it salting is intended to prevent here in the [`__hash__`
documentation][pyhash].

[pyhash]: https://docs.python.org/3/reference/datamodel.html#object.__hash__
