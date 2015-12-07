---
layout: post
title: "Open Source Swift: Asynchronous values revisited"
---

Ten days ago, I wrote a post about [asynchronous values as first-class citizens in Swift](/2015/11/26/async-swift/). Something happened between then and now. Something big, something great: Swift was Open Sourced!

This obviously puts the contents of my previous post in a different light. There's so much to see, read and learn now that everything is in the open. A lot of questions were answered, including the questions I ended the post with:

> The prospect of an open(er) Swift with the (faint) possibility of being able to file a pull request for this really excites me. Of course, there's a long way to go from a blog post to an actual working language feature. It will be so much harder and so much more nuanced than what I've described so far. It's been an amusing thought experiment though and I'd love to learn what it would take to make this reality.

Swift is not just 'open(er)', it is completely open. Code is on GitHub, discussions on mailing lists and bugs are tracked in JIRA. Apple is very serious about making Swift the next big programming language and wants the community to shape, build and own it. I love it! Filing a pull request has become a real possibility.

It comes as no surprise that the Swift engineers at Apple have also been thinking about adding support for asynchronous programming (they use the term 'concurrency') to the language as well. The Swift Programming Language Evolution document however [mentions](https://github.com/apple/swift-evolution/blob/30889943910a4a4e46a800f03d17a91e11ca475f/README.md#out-of-scope) it as being out of scope for the Swift 3.0 release.

To get an idea of how much harder and more nuanced an actual implementation would be compared to what I made up, we can have a look at a [proposal](https://github.com/apple/swift/blob/5eaa3c43d069d5bd401e7879b43f6290823d180d/docs/proposals/Concurrency.rst) by Swift engineer Nadav Rotem.

The proposal primarily describes a thread-safety layer that allows developers to write safe concurrent code and could serve as a foundation for third-party high-level concurrency solutions. When using the proposed thread-safety layer, the compiler can verify the correctness of the code and protect the developer from bugs. This fits nicely with Swift's overall emphasis on safety.

Safety is not guaranteed when directly building on top of something like Grand Central Dispatch, which is exactly what BrightFutures is doing now and which is also the mindset with which I wrote my post. I didn't think big enough when it comes to safety and the low-level features it requires. (Mainly because I haven't got a clue.) Instead, I focussed more on the high-level usage.

# Swift Evolution

So, what's next? There seems to be room for a proposal on a high-level concurrency model for Swift.

The [Swift Evolution Process](https://github.com/apple/swift-evolution/blob/master/process.md#how-to-propose-a-change) guide states that the first step would be to discuss the feature on the swift-evolution mailing list. Next is developing a proposal, followed by a review by a member of the core team. The proposal itself is a pull request on the [swift-evolution repository](https://github.com/apple/swift-evolution). Once it's in there, it can be scheduled to be implemented for a future release.

The whole process seems interesting and exciting, ambitious but also slightly daunting and surely time consuming. I'll have to think about it.
