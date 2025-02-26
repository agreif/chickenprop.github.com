---
title: bsert - "Better" asserts in Python
layout: post
tags: [software]
---
I like Python, but one of the things I dislike is the `assert` statement. Its simplest form provides no help if it fails:

    assert x == y

will raise an exception so you can see *where* it failed, but you don't get to see what `x` or `y` were.

There's a longer form,

    assert x == y, "%r != %r" % (x,y)

but this is verbose, and evaluates `x` and `y` twice. And if `x` and `y` are dictionaries nested three deep, it might not be easy to tell what's different between them.

I'm aware of two approaches that improve the situation. [`nose`](http://nose.readthedocs.org/en/latest/index.html) has a [`--failure-detail`](http://nose.readthedocs.org/en/latest/plugins/failuredetail.html) plugin that tries to automatically give you more detail. When

    assert x == y

fails, it:

1. Finds the source location of the failed assert,
2. Reads and parses this line,
3. Substitutes variables with their values,
4. Reports the substituted line.

This is an *awesome* hack, and I love that it's possible, but I don't find it all that useful. You still need to play spot-the-difference with deeply nested data structures, but that would be pretty easy to fix. The deeper problem is that it also doesn't help with

    assert frobnicate(3) == frobnicate(4)

because there are no variables to replace. (`frobnicate` is a variable, but IIRC it doesn't substitute functions. I don't remember the exact algorithm it uses.) I had a look at the code, and I don't think it would be possible, in general, to report the values on the LHS and RHS. You'd have to re-evaluate the expressions, and there's no guarantee they'd return the same thing the second time.

The second approach is to get rid of `assert` statements completely. In a `unittest` test, you do

    self.assertEqual(x, y)

and if `x != y` it tells you what `x` and `y` are, with a helpful diff format for dicts, lists and sets.

This is great, but I just don't like writing asserts like that. So here's [a new approach](https://github.com/ChickenProp/bsert):

    from bsert import bsert
    bsert | x == y

How it works is that `bsert | x` returns a new object, `_Wrapped(x)`; and `_Wrapped(x) == y` calls `assertEqual(x, y)`. Other comparison methods are overloaded as well. Now we can do things like:

```
$ python
Python 2.7.5 (default, Dec  1 2013, 00:22:45)
[GCC 4.7.3] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from bsert import bsert
>>> bsert | 3 == 3
True
>>> bsert | 3 == 4
Traceback (most recent call last):
  ...
AssertionError: 3 != 4
>>> bsert | [3] + [4] == [3, 4]
True
>>> bsert | [3] + [4] == [3, 4, 5]
Traceback (most recent call last):
  ...
AssertionError: Lists differ: [3, 4] != [3, 4, 5]

Second list contains 1 additional elements.
First extra element 2:
5

- [3, 4]
+ [3, 4, 5]
?      +++

>>> bsert | {1: {2: 3, 4: 5}, 6: 7} == {1: {2: 4, 4: 5}, 6: 7}
Traceback (most recent call last):
  ...
AssertionError: {1: {2: 3, 4: 5}, 6: 7} != {1: {2: 4, 4: 5}, 6: 7}
- {1: {2: 3, 4: 5}, 6: 7}
?         ^

+ {1: {2: 4, 4: 5}, 6: 7}
?         ^

>>> bsert | 1 / 2 != 0
Traceback (most recent call last):
  ...
AssertionError: 0 == 0

>>> bsert | 1.0 / 2 != 0
True
>>> import time
>>> bsert | time.time() != time.time()
True
>>> bsert | time.time() == time.time()
Traceback (most recent call last):
  ...
AssertionError: 1399731667.416066 != 1399731667.416123
>>> bsert | [3] * 3 == [3,3,3]
True
>>> bsert | {1, 2, 3} <= { 1,2,3,4}
True
>>> bsert | {1, 2, 3} >= { 1,2,3,4}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "bsert.py", line 28, in __ge__
    self.assertGreaterEqual(self.wrapped, other)
  File "/usr/lib64/python2.7/unittest/case.py", line 950, in assertGreaterEqual
    self.fail(self._formatMessage(msg, standardMsg))
  File "/usr/lib64/python2.7/unittest/case.py", line 412, in fail
    raise self.failureException(msg)
AssertionError: set([1, 2, 3]) not greater than or equal to set([1, 2, 3, 4])
>>> bsert | 3|8 == 11
True
>>>
```

There are a few limitations. For one, you can't use chained comparisons, and you won't get any kind of error if you try. The reason is that

    bsert | 3 <= 5 <= 4

cashes out as

    (bsert | 3 <= 5) and (5 <= 4)

so there's no way for `bsert` to know that there's another comparison going on. For two, you can't do

    bsert | 3 in [1,2,3]

because there's no way to overload the `in` operator from the left hand side. (In this case, you at least get an `AssertionError: 1 != 3` telling you you did something wrong, because `a in someList` basially does `any(a == x for x in someList)`, and so it fails at `bsert | 3 == 1`. If you had a dict, set, or empty list on the right hand side, it would just return `False` and not raise an exception.)

Similarly, `bsert | x is y`, `bsert | x and y` and `bsert | x or y` don't work, because those operators can't be overridden at all. (Even if they did work, they're low precedence, so it would need to be e.g. `(bsert | x) and y`, which is horrible.) You also can't do

    bsert | False

because that just returns `_Wrapped(False)`

I think all the other operators should work fine, if you're using them in ways that make sense. Most of them have higher-precedence than `|`, so that for example

    bsert | a + b == c

cashes out to

    (bsert | (a + b)) == c

The only exception is `|` itself, and I've added support so that `_Wrapped(x) | y` returns `_Wrapped(x|y)`.

I don't necessarily recommend that you use `bsert`. I'm not sure that I will. But it's there.

I've [put bsert on github](https://github.com/ChickenProp/bsert), but it's also short enough that I might as well just post it inline:

```python
import unittest

class _Bsert(object):
    def __or__(self, other):
        return _Wrapped(other)

class _Wrapped(unittest.TestCase):
    def __init__(self, obj):
        # TestCase needs to be passed the name of one of its methods. I'm not
        # really sure why.
        super(_Wrapped, self).__init__('__init__')
        self.wrapped = obj

    def __eq__(self, other):
        self.assertEqual(self.wrapped, other)
        return True

    def __ne__(self, other):
        self.assertNotEqual(self.wrapped, other)
        return True

    def __le__(self, other):
        self.assertLessEqual(self.wrapped, other)
        return True

    def __ge__(self, other):
        self.assertGreaterEqual(self.wrapped, other)
        return True

    def __lt__(self, other):
        self.assertLess(self.wrapped, other)
        return True

    def __gt__(self, other):
        self.assertGreater(self.wrapped, other)
        return True

    def __or__(self, other):
        return _Wrapped(self.wrapped | other)

bsert = _Bsert()
```

**Belated update:**

On reddit, [Liorithiel informs me](http://www.reddit.com/r/coding/comments/258v7x/bsert_better_asserts_in_python/chev2i5) that [py.test](http://pytest.org/latest/) can extract useful failure messages from `assert` statements. Like what nose does, but implemented differently, so that it can show the values of intermediate expressions in more detail than bsert can. (It rewrites the AST on import time, which is an even more awesome hack than nose's.) As far as I'm concerned, this knowledge makes bsert obsolete.

Meanwhile, obeleh on github has [provided a patch](https://github.com/ChickenProp/bsert/pull/1) which allows bsert to be used with boolean expressions, with different syntax. So

    bsert(3 in [1,2,3])

is like `assert 3 in [1,2,3]`, but with a slightly nicer exception message. (The old syntax still applies for expressions using comparison operators.) Now you get

    AssertionError: bsert(3 in [1,2,3])

instead of just

    AssertionError:

It comes at the cost of a slightly longer traceback - I wonder if that can be worked around. And it doesn't provide intermediate expressions at all, so it's kind of a neat trick, but I don't know if it's useful. (Especially since the old traceback had the failing expression near the bottom anyway - there are cases where you'll see the exception but not the traceback, but we're getting into niche territory.) But that's pretty much how I felt about bsert in the first place, so I decided to go ahead and include it.
