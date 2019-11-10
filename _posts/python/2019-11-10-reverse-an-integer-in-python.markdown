---
layout: post
category: python
date: 2019-11-10
title: Reverse a number (positive integer) in Python
image: /assets/python-wide.png
permalink: /python/reverse-an-integer-in-python
redirect_from: /reverse-an-integer-in-python
---
<div class="wide-logos" markdown="1">
![python](/assets/python.png)
</div>

We are given a number, and we want to reverse it. OK, seems like a fairly
easy problem. Let's take a look at the possible options, both naive and
"smart".

The solutions are given in the form of functions, with the input being a
positive integer.

## Converting an input to string

```python
def reverse_int(i: int) -> int:
    return int(str(i)[::-1])
```

```python
>>> reverse_int(12345)
54321
```

The function will cast the number to a string. Then, you can use slice notation to reverse a number.

Now, the challenge is to reverse a number using only arithmetic.

## Using arithmetic and modulo operation

```python
def reverse_int(i: int) -> int:
    r = 0
    while i > 0:
        r *= 10
        r += i % 10
        i = i // 10
    return r
```

```python
>>> reverse_int(12345)
54321
```

So, the catch here is that we initialize the variable `r` that we will return 
with the 0 as an initial value. Then, as long as the `i` is greater than 0,
expand `r` by a digit (position) and add to it the value of the last digit of `i`. Then, do the integer divison (division followed by a floor function) on `i`. Return `r`.

## Previous solution, with recursion

```python
def reverse_int(i: int, r: int = 0) -> int:
    if i > 0:
        r *= 10
        r += i % 10
        i = i // 10
        return reverse_int(i, r)
    else:
        return r
```

```python
>>> reverse_int(12345)
54321
```

That's it. If you are not familiar with the type hints, we will welcome you
on this [dark side][1]. For the next time, we will include running times for the
problem we are solving, we don't want to break the Internet for the first post.

## Song of the Week

<iframe width="560" height="315" src="https://www.youtube.com/embed/7G1Cv5WuX-c" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

[1]: https://docs.python.org/3/library/typing.html