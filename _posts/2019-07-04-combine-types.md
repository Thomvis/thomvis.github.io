---
layout: post
title: "Why Combine has so many Publisher types"
---

A [question on Twitter](https://twitter.com/pteasima/status/1146065773523161092) prompted me to think about why [Combine](https://developer.apple.com/documentation/combine) has so many implementations of the `Publisher` protocol. There's one for each operator, at least. The [documentation](https://developer.apple.com/documentation/combine/publishers) of the `Publishers` enum lists a whopping 106 implementations. The map operator returns an instance of `Publishers.Map`, filter returns an instance of `Publishers.Filter`, and so on.

All of these derived publisher types refer to their upstream with a generic parameter. Take for example the declaration of `Publishers.Map`:

```swift
struct Map<Upstream, Output>: Publisher where Upstream : Publisher
```

Other operators follow the same pattern. This means that if you create a publisher from an array that you subsequently map and filter, you end up with an instance of `Publishers.Filter<Publishers.Map<Publishers.Sequence<[Int], Never>, Boolean>, Boolean>`. **Except that you don't**. What you'll find is that mapping and then filtering a `Publishers.Sequence` will result in a plain `Publishers.Sequence`.

Combine contains overrides for [map](https://developer.apple.com/documentation/combine/publishers/sequence/3211179-map) and [filter](https://developer.apple.com/documentation/combine/publishers/sequence/3211169-filter) (and many other operators) for `Publishers.Sequence`. Instead of returning a derived publisher that subscribes to the upstream, it creates a new `Publishers.Sequence` that contains the mapped elements. It is able to do so, because in this special case it can access the stored [sequence](https://developer.apple.com/documentation/combine/publishers/sequence/3211222-sequence) in the upstream and use them directly. I imagine the map operator for `Publishers.Sequence` to look something like this:

```swift
extension Publishers.Sequence {
    func map<T>(_ transform: (Elements.Element) -> T) -> Publishers.Sequence<[T], Failure> {
        return Publishers.Sequence(sequence: self.sequence.map(transform))
    }
}
```

The result is a single publisher, instead of a chain of three. I'd call this "fancy compile time chain optimization magic", but people smarter than me have referred to this as "[operator fusion](http://akarnokd.blogspot.com/2016/03/operator-fusion-part-1.html)". This technique is used throughout Combine, not just for `Publishers.Sequence`.

Having this many concrete publisher types is only feasible because of Swift 5.1's [opaque return type](https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html). But even then, types can get in the way. For those cases there's `AnyPublisher`, which wraps any publisher to erase its specific type. But now you know that it can close the door on some pretty cool optimizations as well.
