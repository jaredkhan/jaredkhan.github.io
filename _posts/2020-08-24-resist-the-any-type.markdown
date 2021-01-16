---
layout: post
title:  "Python Typing: Resisting the Any type"
date:   2020-08-29 16:29:00 +0100
tags: Python
---

*With typing in Python, we aim to restrict as many invalid programs as possible before they're ever run.
This post covers some useful features for tightening up our types*:

- `TypeVar` for when an unknown type appears multiple times in the same context [[Jump]](#polymorphism-with-typevar)
- `@overload` for when you have a function whose behaviour depends on its input [[Jump]](#overloading-with-overload)
- `Protocol` for supporting any type with the desired attributes and methods [[Jump]](#static-duck-typing-with-protocol)

## Introduction

Python itself doesn't have a static type checker, but does provide a syntax for adding type annotations to your code: 

```python
# basics.py

from typing import List
from dataclasses import dataclass


# We can annotate the input and output types of functions.
def fibonacci(n: int) -> int:
    assert n >= 0
    if n == 0:
        return 0
    if n == 1:
        return 1
    return fibonacci(n - 1) + fibonacci(n - 2)


@dataclass
class Document:
    # We can annotate class attributes too.
    # We're using the generic type 'List' with parameter 'str'.
    # This means the `lines` attribute is a list of strings.
    lines: List[str]


# This is invalid:
broken_doc = Document(lines=[1, 2, 3])

# This is valid:
fibonacci_doc = Document(
    lines=[
        f"{n}th fib number is {fibonacci(n)}"
        for n in range(20)
    ]
)
```

IDEs like [PyCharm](https://blog.jetbrains.com/pycharm/) and type-checking tools like [Mypy](http://mypy-lang.org) and [Pyre](https://pyre-check.org) use these annotations to identify type errors. The intended rules of this type-checking are mostly standardised in Python Enhancement Proposals (PEPs), so the behaviour is similar between the tools. 

### Getting started with Mypy

Here, we'll use Mypy for type-checking. To get started with Mypy, simply `pip install mypy`. 
Here's an example of running Mypy using the basics.py file from above 

```bash
# In your terminal:
virtualenv mypy-venv -p python3.8
source mypy-venv/bin/activate
pip install mypy
mypy --version
# mypy 0.770
mypy --strict python-typing-playground/basics.py
# warm-up/basics.py:22: error: List item 0 has incompatible type "int"; expected "str"
# warm-up/basics.py:22: error: List item 1 has incompatible type "int"; expected "str"
# warm-up/basics.py:22: error: List item 2 has incompatible type "int"; expected "str"
# Found 3 errors in 1 file (checked 1 source file)
```

I'm using Python 3.8. Many interesting typing features were added to the standard library in 3.8 but if you can't use Python 3.8, [`typing-extensions`](https://pypi.org/project/typing-extensions/) often has your back.

Mypy lets you get started with a large existing untyped code base and gradually add type hints over time. 
It will simply silently treat all objects as `Any` if it can't find type hints for them.
I recommend using the [`--strict` flag](https://mypy.readthedocs.io/en/stable/command_line.html#cmdoption-mypy-strict) when first trying it out. 
This mostly stops Mypy from silently assuming things are `Any`, which I think is helpful when trying to understand what Mypy can and cannot infer. 

### What's wrong with `Any` anyway?

Adding types restricts the way our functions, classes and variables can be used. This restriction is good because it helps catch whole categories of bugs before the code is ever run whilst also (usually) making the code easier to think about. Typically, our goal with typing in Python is:
 - to restrict as many 'invalid' usages as possible,
 - whilst allowing all 'valid' usages,
 - and keeping our code relatively tidy.
 
Sometimes, using only the syntax given above, we might get stuck on how to express the type of something and turn to the `Any` type.
The `Any` type is a magical type which pretends to the type-checker to support any operation you could possibly want, and to also be a parent type of all types:

```python
from typing import Any


def do_something_shady(magic_input: Any) -> Any:
    # We can do any operation we want on an Any object.
    # The type-checker will allow it 
    # and assume the result is an Any object
    return magic_input.thing["stuff"] + 42


# All types are compatible with Any,
# so even if we pass in a str, this is valid.
# If we try to run this, it will fail,
# since a string doesn't have a 'thing' attribute.
do_something_shady(magic_input="just a string")
```

Clearly, using `Any` can allow many invalid programs. 
I'd like to share a few useful Python type-system features that we can consider before giving in to the `Any` type's magical allure, or otherwise loosening our types.


## Polymorphism with `TypeVar`

Type variables lets us specify that some unknown type will appear multiple times in the same context, 
whether that be multiple times in the same function signature, or multiple times in methods of a generic class.
We don't know what the type will be, but we know it will be the same in all the places where it appears.


```python
from typing import List, TypeVar
from dataclasses import dataclass

# Declare a type variable
ValueType = TypeVar("ValueType")  # The string in this constructor must match the variable name


def repeat(value: ValueType, count: int) -> List[ValueType]:
    """Returns the input value repeated `count` times in a list."""
    return [value] * count

# This is valid
", ".join(repeat("a", count=5))

# This is invalid (can't do a string join on a list of ints)
", ".join(repeat(3, count=5))
```

Because `ValueType` is used in two different places in the same signature, Mypy will check, for each call to this function,
that the values used in those two places match types. 
In this case, it checks that the type of value returned from the function is always a list of the type of value given to the function. 
This also means Mypy knows more about the result of calling the function, and can be stricter there too.

### Using bound=

So `TypeVar` is great for capturing this idea of an *unknown* type appearing multiple times.
Sometimes we do know something about the type, and maybe we want to use a certain field that we know will exist on the type,
but we still want to capture and check the fact that it appears multiple times.
 
By default a TypeVar will bind to any type, all the way up to `object` but we can put an 'upper bound' on this using the `bound` parameter.
This says 'only let this TypeVar bind to a subtype of X', where X is some type that we care about. 

```python
from typing import List, TypeVar
from dataclasses import dataclass

@dataclass
class Animal:
    name: str


class Dog(Animal):
    def pat(self) -> None:
        print(f"gave {self.name} a good pat")


class Cat(Animal):
    def stroke(self) -> None:
        print(f"stroked {self.name}")


AnimalType = TypeVar('AnimalType', bound=Animal)


def sort_animals_by_name(items: List[AnimalType]) -> List[AnimalType]:
    return sorted(items, key=lambda animal: animal.name)


# This is valid (all the inputs are dogs so all have the 'pat' method)
sorted_dogs = sort_animals_by_name([Dog(name="spots"), Dog(name="buzz")])
sorted_dogs[0].pat()

# This is invalid
sorted_dogs[0].stroke()

# This is invalid (cannot use str due to bound=Animal)
not_animals = ["John", "Joan", "Jan"]
sort_animals_by_name(not_animals)
```

Because we specified that `AnimalType` must be a subtype of `Animal`, the type-checker allows us to use the `name` property from `Animal` within our function.
Note that we still aren't able to use a more specific method like `pat` within the body of `sort_animals_by_name`, since it doesn't always exist on the upper bound, `Animal`.

### Misconceptions about scoping

Because of the strange syntax for declaring a type variable, where we create a global object, it's easy to get confused about their scoping rules:

**Question: Is the following valid?**

```python
from typing import List, TypeVar

T = TypeVar("T")

def repeat(value: T, times: int) -> List[T]:
    return [value] * times

def identity(x: T) -> T:
    return x

repeat("echo", times=5)
repeat(42, times=5)
identity(42)
```

After all, we seem to be asking `T` to be a string and then to be an int.

The answer is **yes, this is valid**. 
Generic functions using TypeVars can happily be called multiple times with different input types. 
The two calls to the function are unrelated as far as this TypeVar is concerned.
Similarly, TypeVars can happily be recycled across multiple independent functions. 
You often only need multiple TypeVars if you need multiple distinct types within the _same_ function signature.

See [Scoping rules for type variables within PEP-484](https://www.python.org/dev/peps/pep-0484/#scoping-rules-for-type-variables) for more examples.


## Overloading with `@overload`

Sometimes we have an *overloaded* function, one which has different behaviour depending on the input given.

Overloading in Python is quite different to other languages. 
In other languages, we might write multiple function implementations with the same function name but different arguments (either in type or in name)
and when that function is called, the correct implementation would be chosen based on the given arguments.

In Python, we are not going to define multiple implementations 
(remember our type annotations aren't read at runtime). 
In Python we just have one implementation, which manually checks the types and does the right thing.
All `@overload` lets us do is tell the *type checker* which combinations of parameters and outputs are valid.

We do this by providing multiple `@overload` signatures before defining our actual implementation.


```python
from typing import Optional, overload
import math

@overload
def get_circle_area(*, radius: float) -> float: ...

@overload
def get_circle_area(*, circumference: float) -> float: ...

def get_circle_area(*,
     radius: Optional[float] = None,
     circumference: Optional[float] = None
) -> float:
    """
    Takes either a radius or circumference of a circle,
    and returns the area of that circle.
    """

    # Check we've been given exactly one of the two forms of input
    if radius is not None and circumference is not None:
        raise ValueError("Can't use both radius and circumference")
    elif radius is not None:
        canonical_radius = radius
    elif circumference is not None:
        canonical_radius = circumference / math.tau
    else:
        raise ValueError("Give either a radius or circumference")
    
    return math.pi * canonical_radius * canonical_radius


# Invalid (want exactly one of radius or circumference):
get_circle_area()
get_circle_area(radius=None)
get_circle_area(circumference=None)
get_circle_area(radius=1.0, circumference=None)
get_circle_area(radius=1.0, circumference=3.0)
get_circle_area(3.0)  # ambiguous without the argument keyword

# Valid:
get_circle_area(radius=1.0)
get_circle_area(circumference=math.tau)
```

We used two `@overload` signatures (with empty implementations) before specifying the implementation itself.
The type signature of the implementation itself is only used for type-checking that implementation, not for type-checking usages of the function.
The type checker will also make sure that our implementation supports all the `@overload` signatures described. 
For example, If we have an `@overload` which accepts a string argument, but the type of the implementation only takes floats, Mypy will throw an error. 

As a side note, calling this function with a number but without the argument keyword (e.g. `radius=`) would be ambiguous, we wouldn't know if the number was a radius or a circumference.
The asterisk in the function signatures enforces that keywords be used.
For more explanation, see [Keyword-Only Arguments — Specification](https://www.python.org/dev/peps/pep-3102/#specification)


## Static Duck Typing with `Protocol`

If we're restricting the input types of a function, we sometimes only care that it has certain attributes and methods,
not whether it's a _subclass_ of a particular class. 
Furthermore, it's not always possible to modify a class to inherit from a certain base class,
 because it might be from a library that you don't control.

`Protocol` lets us capture these attribute and method constraints without requiring the type to inherit from any particular class.

```python
# If you can't use Python 3.8, you can also import Protocol from typing-extensions
# https://pypi.org/project/typing-extensions/
from abc import abstractmethod
from typing import Protocol
from dataclasses import dataclass

# Setup: we'll define a few classes which happen to have a 'name'

@dataclass
class Person:
    first_name: str
    last_name: str
    height: float

    @property
    def name(self) -> str:
        return self.first_name + " " + self.last_name

    def greet(self) -> str:
        return f"Hi, my name is {self.name}."


@dataclass
class Company:
    name: str
    address: str

    def greet(self) -> str:
        return f"We are {self.name}, find us at {self.address}."


@dataclass
class Dog:
    name: str
    color: str

    def greet(self) -> str:
        return "Woof!"



class Greetable(Protocol):
    """
    Matches any type with
    a readable (but not necessarily writeable) `name`
    and a `greet` method that returns a string.
    """
    @property
    def name(self) -> str: ...

    def greet(self) -> str: ...


def introduce(greetable: Greetable) -> None:
    print(
        f"This thing is named {greetable.name}"
        f" and it says '{greetable.greet()}'"
    )


class Renameable(Protocol):
    """Matches any type with a read/write `name`."""
    name: str


def rename(renameable: Renameable, new_name: str) -> None:
    renameable.name = new_name


spots = Dog(name="spots", color="white")
acme = Company(name="Acme Inc.", address="123 Industry Ave.")
john = Person(first_name="John", last_name="Johnson", height=1.96)

# Valid things
introduce(spots)
introduce(acme)
introduce(john)
rename(spots, "spotsy")

# Invalid things
introduce(42)
introduce("blah")
rename(john, "J-man")  # Person's `name` is read-only
```

None of the classes we defined inherit from `Greetable` (in fact, non-protocol classes cannot inherit from Protocols)
yet they all conform to it because they have a `name` and a `greet` method with the appropriate types.
As such, they can all be passed to a function that takes a `Greetable`.

When specifying attributes that must exist on the type, we saw both:
- the `@property` form, which lets us specify that we only need to read this value,
- and the `name: str` form which is a shorthand for an attribute that is both readable and writeable.

We can also opt in to these Protocols being checkable at runtime. For more detail, see [@runtime_checkable decorator within PEP 544](https://www.python.org/dev/peps/pep-0544/#runtime-checkable-decorator-and-narrowing-types-by-isinstance)


## Conclusion

In this post we've seen some useful features for tightening our types:

- Use `TypeVar` to express that a single unknown type will appear multiple times in the same context
- Use `@overload` when you have a function whose behaviour depends on its input
- Use `Protocol` when you want to support any type which happens to have the attributes and methods that you need.
