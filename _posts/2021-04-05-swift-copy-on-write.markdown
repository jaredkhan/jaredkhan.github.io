---
layout: post
title:  "Swift's Copy-on-write Optimisation"
date:   2021-04-05 12:45:00 +0100
tags: Swift
---

Swift's Arrays have value semantics. They also have a copy-on-write optimisation: 

> Collections defined by the standard library like arrays, dictionaries, and strings use an optimization to reduce the performance cost of copying. Instead of making a copy immediately, these collections share the memory where the elements are stored between the original instance and any copies. If one of the copies of the collection is modified, the elements are copied just before the modification. The behavior you see in your code is always as if a copy took place immediately.

[https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html#ID88](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html#ID88)

Here's a few points to help us understand how the copy-on-write optimisation works:

- The Array type is a value type.
- Array has a stored property which is a **reference** to a buffer object which is where the Array data actually lives (on the heap).
- When an Array is copied, the copy gets a reference to the same buffer. The reference count of the buffer has now increased.
- When a mutation is applied to an Array, the reference count of the buffer is checked. If it is greater than 1, the buffer is copied first. Otherwise, the buffer is not copied. In this sense it's not just copy-on-write but copy-on-write-when-necessary.
- This behaviour is hand-written in the Standard Library code for Array and other container types, it's **not** an automatic feature of Swift value types.

With this in mind we should be able to reason about the memory performance of the following snippets. I've used pretty large numbers throughout, so that it's easy to see the changes in memory usage in Activity Monitor (or other tools).

I've put these examples into [copy_on_write_examples.swift](https://gist.github.com/jaredkhan/5c77a7f79c6adcdf58c9a08c9d53a023) in case you want to run them for yourself.

See if you can predict what the approximate memory usage will be at each of the numbered comments in these snippets.

### Copying a large array

```swift
var x = Array(repeating: Int64(1), count: 100_000_000)
// Memory usage is ~800MB
var y = x
// (1)
y.append(2)
// (2)
```

This is a fairly simple case.

At (1) we have taken a copy of x into y but not yet mutated it so at this point we still just have a reference to the same buffer, memory usage is still around 800MB

At (2) we have mutated y's data so Swift will notice that the buffer is not uniquely referenced and will copy it fully before the mutation. Memory usage is now around 1600MB

### Nested Arrays

```swift
var x = [
	Array(repeating: Int64(1), count: 100_000_000),
	Array(repeating: Int64(1), count: 100_000_000),
]
// Memory usage is ~1600MB
var y = x
y.append([])
// (1)
y[0].append(2)
// (2)
```

There's a little bit of nesting going on here.

This time, x's buffer is not storing a lot of data, rather it is storing 2 small Array structs which themselves have references to large buffers. After we take a copy of x and then mutate its buffer, it will copy x's buffer but won't copy the buffers of x's elements.

At (1), memory usage is still around 1600MB.
At (2), we've made a mutation to one of the large buffers, so that one buffer will be copied first, so we expect a memory usage of about 2400MB


### Array within a struct

```swift
struct ThingWithArray {
	let name: String
  let array: [Int64]
}

let x = ThingWithArray(
  name: "Nice Thing",
  array: Array(repeating: Int64(1), count: 100_000_000)
)
// Memory usage is ~800MB
var y = x
// (1)
y.name = "Nicer Thing"
// (2)
y.array.append(1)
// (3)
```

This is similar to the last case. The struct itself doesn't hold much data, the `array` property just holds a reference to a buffer which stores a lot of data.

At (1) the struct is copied into y, so y now has a reference to the same array buffer and the memory usage is still around 800MB.

At (2), mutating y doesn't touch the array buffer, so still no big change in memory usage.

At (3), the array buffer is mutated via y so the buffer is now copied and memory usage goes to about 1600MB.

### Many structs

```swift
  struct Thing {
    // A struct with about 80 bytes
    let a: Int64 = 0
    let b: Int64 = 0
    let c: Int64 = 0
    let d: Int64 = 0
    let e: Int64 = 0
    let f: Int64 = 0
    let g: Int64 = 0
    let h: Int64 = 0
    let i: Int64 = 0
    let j: Int64 = 0
  }
  let thing = Thing()
  let x = Array(repeating: thing, count: 10_000_000)
  // (1)
```

Structs themselves do not have automatic copy-on-write semantics, so taking many copies of a simple struct, even if they are not mutated, will really cause them to be copied. Memory usage at (1) is about 800MB.

### Mutating when uniquely referenced

```swift
  var x = Array(repeating: Int64(1), count: 100_000_000)
	// Memory usage is ~800MB
	if true {
    let y = x
  }
  x.append(1)
  // (1)
```

Here we copy x into y but then y goes out of scope so the reference count on the large array buffer drops back down to 1. At (1), when we've mutated the array, the reference count is still 1 and so no copy is taken and the memory usage is still ~800MB.

## References and Further Reading

- [Wikipedia — Value Semantics](https://en.wikipedia.org/wiki/Value_semantics)
- [swift/docs/OptimizationTips.rst — Use copy-on-write semantics for large values](https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst#advice-use-copy-on-write-semantics-for-large-values)
    - This discusses how to add copy-on-write behaviour to your own value types using isKnownUniquelyReferenced
- [Ben Cohen - Fast Safe Mutable State](https://www.youtube.com/watch?v=BXJIIQ-B4-E)
    - Slightly related, manager of the Swift Standard Library team explains copy-on-write and some interesting cases where the Standard Library takes extra care to prevent buffers being referenced multiple times unnecessarily.
- [swift/stdlib/public/core/Array.swift](https://github.com/apple/swift/blob/main/stdlib/public/core/Array.swift)
    - [Declaration of the buffer reference](https://github.com/apple/swift/blob/main/stdlib/public/core/Array.swift#L310)
    - [The method that is called before every mutating method to copy the buffer if necessary](https://github.com/apple/swift/blob/main/stdlib/public/core/Array.swift#L347)
    - [The definition of the buffer](https://github.com/apple/swift/blob/main/stdlib/public/core/ContiguousArrayBuffer.swift#L256)
