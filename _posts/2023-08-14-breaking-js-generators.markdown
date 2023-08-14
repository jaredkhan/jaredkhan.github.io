---
layout: post
title: "Breaking JavaScript Generators"
date: 2023-08-14 10:00:00 +0100
tags: Javascript Python
---

_Coming back to JavaScript after focusing on Python, I was surprised that using `break` when iterating over a generator closes the generator. This post has an example of that behaviour and a comparison with Python's behaviour._

Python and JavaScript both have the concept of generators, functions which can yield multiple values, pausing after each yield until the consumer asks for the next value.

```python
# Python
def my_generator():
	yield 1
	yield 2
	yield 3

iterator = my_generator()

print(iterator.next())
print("something else")
for item in iterator:
  print(item)

# Outputs:
# 1
# something else
# 2
# 3
```

```js
// Javascript
function* myGenerator() {
  yield 1;
  yield 2;
  yield 3;
}

const iterator = myGenerator();

console.log(iterator.next().value);
console.log("something else");
for (const item of iterator) {
  console.log(item);
}

// Outputs:
// 1
// something else
// 2
// 3
```

These two things look basically identical. The iterator object we get back in the two cases has pretty much the same three main functions:

- `__next__`(Python)/ `next`(JavaScript)
  - Run the generator up to the next yield and get the yielded value
- `throw(exception)`
  - Force the generator to throw the given exception from its current suspended position
- `close`(Python)/`return(value)`(JavaScript)
  - Close the generator, making sure any `finally` blocks are run. Can’t get values out of this generator after this (unless there are `yield`s in the `finally` for some reason).

The interesting difference that surprised me is that in JavaScript, calling `break` whilst iterating over a generator calls `return` on the generator, so you can’t really keep using that generator:

```python
# Python
def my_generator():
  try:
    yield 1
    yield 2
  finally:
    print("finally")

it = my_generator()

# Print one item then break
for item in it:
  print(item)
  break

# Print the rest of the items
for item in it:
  print(item)

# Outputs:
# 1
# 2
# finally
```

```js
// Javascript
function* myGenerator() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("finally");
  }
}

const it = myGenerator();

// Print one item then break
for (const item of it) {
  console.log(item);
  break;
}

// Try to print the rest of the items
// (doesn't work)
for (const item of it) {
  console.log(item);
}

// Outputs:
// 1
// finally
```

In Python, the `close` gets called in the `__del__` of the iterator object. That is, it gets closed when it gets garbage collected. This isn’t really possible in JavaScript due to the lack of any standardised garbage collection scheme, so I guess they had to do something else to make sure ‘finally’ is more likely to actually be called. That said, if it’s iterated outside of a `for..of`, it’s easy enough to just not close it and the finally will never run.

```js
// Javascript
function* myGenerator() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("finally");
  }
}

const iterator = myGenerator();
console.log(iterator.next().value);

// Outputs:
// 1
// (never outputs 'finally')
```

**Further Reading**

- [PEP 342 – Coroutines via Enhanced Generators](https://peps.python.org/pep-0342/#specification-summary)
  - This specifies the behaviour of the `try..finally` in Python generators
- [MDN – for...of](<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of#description:~:text=If%20the%20for...of%20loop%20exited%20early%20(e.g.%20a%20break%20statement%20is%20encountered%20or%20an%20error%20is%20thrown)%2C%20the%20return()%20method%20of%20the%20iterator%20is%20called%20to%20perform%20any%20cleanup>)
  - This mentions the `break` behaviour
