---
layout: post
title: "Asynchronous values as first-class citizens in Swift"
---

Wouldn't it be great if Swift made some common tasks in modern app development easier? Things like networking and other asynchronous operations, user interface and interaction definitions and working with JSON. Swift hasn't necesarily made these things easier, compared to Objective-C.

Luckily, Swift is powerful enough to allow third-party developers to create frameworks that make the aforementioned tasks easier in a way that integrates nicely with the language and the standard library. Some examples: [Carthography](https://github.com/robb/Cartography), [Few](https://github.com/joshaber/Few.swift), [Argo](https://github.com/thoughtbot/Argo) and [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa).

In this post, I'll take a look at asynchronous operations. I specifically like a particular approach towards this: Futures. I like them so much that I've written my own implementation ([BrightFutures](https://github.com/BrightFutures)) and recently gave a [talk](http://www.thomasvisser.me/2015/11/08/swiftsummit-execution-context/) on the subject at SwiftSummit. At the conference, I was also on a panel that discussed the impending open sourcing of Swift. This post combines those two topics into a story that begins with how  third party libraries are already improving asynchronous programming and what more can they do without needing changes in the language. The second part of the post is on how I imagine built-in support for asynchronous values using Futures could look like.

# A Third-party Future

I'm going to assume a basic understanding of what Futures are, but I'll provide the following short explanation, courtesy of the [Scala documentation](http://docs.scala-lang.org/overviews/core/futures.html):

> A Future is a placeholder object for a value that may not yet exist. Generally, the value of the Future is supplied concurrently and can subsequently be used. Composing concurrent tasks in this way tends to result in faster, asynchronous, non-blocking parallel code.

Consider the following example:

{% highlight swift %}
let image = fetchImageBaseUrlForItem(item).map  { baseUrl in
    return url + "?w=1920&h=1080"
}.map { url in
    return session.fetchImage(url)
}.map { img in
    return img.imageFlippedForRightToLeftLayoutDirection()
}
// img is a Future<UIImage, E>
{% endhighlight %}

You've probably written quite a lot of code that does something similar to this: retrieve an url to an image from the network, add dimensions to the image url, fetch the image and flip it for RTL languages. `fetchImageBasedUrlForItem` returns a `Future<String,E>`, a *future string*, a placeholder for a value that we're waiting for to arrive over the network. Meanwhile we can describe the operations that will happen as soon as the value comes in. The chain of operations as a whole returns a Future that represents the result of the final step.

This is the kind of code you can write today with the help of BrightFutures. While it is an improvement over the vanilla solution (i.e. completion handlers, which tend to lead to code that looks like a [pyramid](https://github.com/Thomvis/SFSwiftSummit2015/blob/9f38ae6fa65b31540f2b1ebd110965c51bb38690/SwiftSummit/DataSource.swift#L39-L59)), it also adds some noise. All changes are wrapped in `map` closures. 

We can do a bit better, even without having to change Swift itself. We can define an alternate version of the append operator (`+`) that works on a Future string and a regular string and returns a Future string:

{% highlight swift %}
func +<E>(lhs: Future<String, E>, rhs: String) -> Future<String, E> {
    return lhs.map { l in
        return l + rhs
    }
}
{% endhighlight %}

This would allow us to write the following:

{% highlight swift %}
let url = fetchImageBaseUrlForItem(item) + "?w=1920&h=1080"
// url is a Future<String, E>
{% endhighlight %}

This removes all visual overhead that comes with Futures and leaves us with a seemingly boring concatenation of two regular strings. The good kind of boring.

The example above continues by fetching an image that is located at the url that we've just built. I'm using `fetchImage(_:)` which takes a URL string returns a Future image found at that url. It's a wrapper around the `dataTaskWithURL(_:_:)` in an extension on NSURLSession and looks something like [this](https://gist.github.com/Thomvis/dc9cae1ff295dc7176b6). To be able to write this part as cleanly as the concatenation, we need a version of `fetchImage(_:)` that takes a Future string a parameter. We can add this to session by creating an extension on `NSURLSession`:

{% highlight swift %}
extension NSURLSession {
    func fetchImage<E>(url: Future<String, E>) -> Future<UIImage, E> {
        return url.flatMap { str in
            return self.fetchImage(str)
        }
    }
}
{% endhighlight %}

Next up is the flipping of the image. Here the asynchronous value, the image, is the target of the function call. But since we're dealing with a Future image, instead of a regular image, we need to redefine `imageFlippedForRightToLeftLayoutDirection()` on Future images:

{% highlight swift %}
extension Future where Value: UIImage {
    func imageFlippedForRightToLeftLayoutDirection() -> Future<UIImage, E> {
        return map { img in
            return img.imageFlippedForRightToLeftLayoutDirection()
        }
    }
}
{% endhighlight %}

We can now rewrite the example that looks really clean:

{% highlight swift %}
let url = fetchImageBaseUrlForItem(item) + "?w=1920&h=1080"
let image = session.fetchImage(url)
                   .imageFlippedForRightToLeftLayoutDirection()
// image is a Future<UIImage, E>
{% endhighlight %}

While it would be possible to programmatically generate Future-compatible versions of all built-in operators and functions, I think we could go a lot further if the language would have support for asynchronous values.

For the remainder of this post, I'd like to think about what built-in support for asynchronous values would look like. Needless to say, anything beyond this point is merely a thought experiment, a mental preparation for a pull request that I'd love to file once Swift is open sourced. **It's Swift fan fiction.**

# In a Swift version far far away...

It seems to me that in order to have Futures as first-class asynchronous values in Swift nothing really revolutionary is needed. Parts of it are alreay there; the syntactic sugar of Optionals and the way NSError inout parameters lead to throwing functions have paved the way. 

There's the `Optional` enum, but you hardly ever use it directly. At the same time, almost all functionality of the optional type would work without language support. I really like this approach and would love to see a powerful Future type in the standard library that is boosted by special treatment by the compiler.

Let's imagine what this would look like.

Firstly we need a way to mark a type as asynchronous, like the question mark for optionals. Let's use the `async` keyword and allow it to be put in front of a type. If it is used in the return type of a function, the function becomes an asynchronous operation. Any functions called on an `async` type also become asynchronous operations.

In the context of the example that we've been using so far, let's look at how `fetchImage(_:)` can be implemented:

{% highlight swift %}
extension NSURLSession {
    func fetchImage(url: String) -> async UIImage {
        let data, response = dataTaskWithURL(url)
        return UIImage(data: data)
    }
}
{% endhighlight %}

The compiler would be able to automatically generate a `dataTaskWithURL(_:)` that returns an `(async NSData, async NSURLResponse)` tuple because it could automatically wrap the method with the same name that takes a `completionHandler` as the final parameter, similar to how NSError inout parameters result in throwing functions.

By passing the async data into the UIImage initialiser, the initialiser becomes asynchronous as well. This could happen automatically as you pass asynchronous values to methods that expect their synchronous counterparts.

For a lot of functions, operators included, the compiler could just generate asynchronous variants on the fly. For other situations, e.g. when there's IO involved, there should be a way to turn an asynchronous type into a regular type:

{% highlight swift %}

let asyncImage = session.fetchImage(url)
                        .imageFlippedForRightToLeftLayoutDirection()

when let image = asyncImage else {
    throw ImageLoadingFailed()
}

imageView.image = image
{% endhighlight %}

The `when let` statement pauses the execution of the current scope until the asynchronous string value has been resolved (i.e. fetched from the network). If the operation failed, the `else` clause is executed. If successful, the execution will continue with a regular UIImage value in `image`. While the execution is paused, control is returned to the previous frame on the stack. The return type of the scope that contains the `when let` statement has to be marked `async`. If it does not have a return type, it has to be marked as `async Void`.

To check if an asynchronous value has already been resolved, optional*esque* syntax could be used: `if let image = asyncImage { ... }`, `asyncImage?.size` as well as force unwrapping an asynchronous value using an exclamation mark, risking a runtime exception if the value is not yet there.

The prospect of an open(er) Swift with the (faint) possibility of being able to file a pull request for this really excites me. Of course, there's a long way to go from a blog post to an actual working language feature. It will be so much harder and so much more nuanced than what I've described so far. It's been an amusing thought experiment though and I'd love to learn how far fetched it is.

*Thanks to [@nielsify](https://twitter.com/nielsify) and [@larslockefeer](https://twitter.com/larslockefeer) for providing feedback on a draft of this post.*
