---
layout: post
title: "Handcrafted artisanal random identifiers"
---

How do you make sure related code in different places is kept in sync?

One solution would be to refactor your code so the related pieces are put together. But let's not always try to solve our problems by improving our code base, shall we? 🙃

The other solution that I've been using is to group these places together by adding a shared unique identifier in a comment at each location. Whenever you make a change in one of these places, you'll notice the identifier, do a project search for that identifier and also update the code that comes up in the search results.

Here's an example of an identifier in code:

```swift
// Outcomes should be kept in sync with Future states (see 9s07dvyhs0r)
func materialize<E>(_ f: ((E?) -> Void) -> Void) -> Future<Void, E> {
    // implementation omitted
}
```

This identifier is pretty unique. It has 0 results on Google.

Searching for it in the project looks like this:

![Xcode search panel](/media/artisanal-identifiers-search.png)

There are multiple ways you can create those identifiers (i.e. identifier creation strategies), but I prefer handmade artisanal random identifiers. These are created by mashing buttons on your keyboard until a reasonably sized, random looking string has appeared at your cursor. The chances of coming up with the same identifier twice are... very small. Let's leave the exact calculations to another post. Also take care not to [end up](https://en.wikipedia.org/wiki/Infinite_monkey_theorem) with the complete works of William Shakespeare.

I'm not sure if this is quite brilliant or very stupid, so do let me know.