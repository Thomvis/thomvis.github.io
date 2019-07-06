---
layout: post
title: "Combine's Future"
---

As the author of [BrightFutures](https://github.com/Thomvis/BrightFutures), a somewhat popular implementation of Futures & Promises in Swift, I felt obliged to take a look at Combine's `Future` type and see what first party support for these concepts is like.

Futures and Publishers (or [reactive streams](https://www.reactive-streams.org)) are both abstractions for asynchronous programming. There are a few key differences though:

- Futures represent the result of an asynchronous operation. Publishers represent the operation as a whole.
- Futures emit at most one value or an error. Publishers emit any number of values followed by an error or completion event.
- A Future's asynchronous operation is executed once, regardless of the amount of subscribers. A Publisher doesn't share its operation between subscribers by default, but can be configured to do so.

Let's look at an example using `Future`:

```swift
func task() -> Future<Int, Never> {
    return Future { promise in
        print("Perform operation...")
        // Do some work...
        promise(.success(42))
    }
}

let f = task()

print("Let's sink")
f.sink { v in
    print("sink 1: \(v)")
}

print("Let's sink again")
f.sink { v in
    print("sink 2: \(v)")
}

// Prints:
// Perform operation...
// Let's sink
// sink 1: 42
// Let's sink again
// sink 2: 42
```

The closure passed to `Future`'s initializer is immediately invoked, so when the future is assigned to `f`, the operation has already started (and, in this case, already finished since we're doing no asynchronous work.) Each `sink` receives the result of the same, single execution of the operation.

Let's compare this with a Publisher[^1]:

```swift
func task() -> AnyPublisher<Int, Never> {
    return AnyPublisher { subscriber in
        print("Perform operation...")
        // Do some work...
        subscriber.receive(42)
        subscriber.receive(completion: .finished)
    }
}

let f = task()

print("Let's sink")
f.sink { v in
    print("sink 1: \(v)")
}

print("Let's sync again")
f.sink { v in
    print("sink 2: \(v)")
}

// Prints:
// Let's sink
// Perform operation...
// sink 1: 42
// Let's sink again
// Perform operation...
// sink 2: 42
```

Here you see that the operation is executed twice, once for each subscriber. The closure passed to `AnyPublisher` is not called upon initialization, as we saw with a Future, but at the moment of subscription instead. A Publisher is lazy or, in Rx parlance, [_cold_](https://github.com/ReactiveCocoa/ReactiveSwift/blob/master/Documentation/RxComparison.md#signals-and-signalproducers-hot-and-cold-observables) by default.

# Why use Futures if you have Publishers?
You might have noticed from the list of differences above that a publisher can do anything a future can, and more. To put it otherwise, Publishers are a superset of Futures. Combine makes this very clear: `Future` implements the `Publisher` protocol. It's one of many publisher implementations, but happens to be one with a familiar name and behavior.

Some operations might be better represented by a Future, so it's nice that we have the option to return something with that name and not just a generic Publisher that behaves like one. The Future type exposes this common behavior to the type system and can enforce it through type safety. On the other hand, when you go all in on reactive programming, you might find that most of your code does not make assumptions on the behavior of a Publisher, how many values and when it might emit.

In any case, there is something left to be desired of Combine's `Future`. Any operation called on a `Future` returns a `Publisher`, even the ones that wouldn't change the behavior such as `map`. The `Future` instance is wrapped and hidden, the behavior no longer apparent at a glance. Apple [overloads operators](/2019/07/04/combine-types/) in other parts of Combine to prevent this, but it appears `Future` hasn't received the same treatment yet.

[^1]: The `AnyPublisher` initializer that takes a subscribe closure disappeared in beta 3. Let's hope it comes back.