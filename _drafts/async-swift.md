---
layout: post
title: "Asynchronous values as first-class citizens in Swift"
---

Swift is designed to be a great programming language for a lot of things. On the first page of the Swift Programming Language book, you'll read the following:

> The compiler is optimized for performance, and the language is optimized for development, without compromising on either. It’s designed to scale from “hello, world” to an entire operating system.
>
> [link](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0)

The reality however is that Swift will be mostly used for iOS and Mac app development. This could change of course when Swift is open sourced, but for the Apple platforms it will be the primary and (eventually) only language of choice.

Wouldn't it be great if Swift made it easier to do some common tasks in modern app development. Things like networking and other asynchronous operations, user interface and interaction definitions and working with JSON. Swift has nothing that makes those tasks easier than when we were still writing Objective-C.

Luckily, Swift is powerful enough to allow third-party developers to create frameworks that make the aforementioned tasks easier in a way that the solution is integrated nicely in the language. Some examples: [Carthography](https://github.com/robb/Cartography), [Few](https://github.com/joshaber/Few.swift), [Argo](https://github.com/thoughtbot/Argo), [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) and [BrightFutures](https://github.com/BrightFutures).

In this post, I'd like to focus on the asynchronous operations: how third party frameworks are already improving this and how it could be incorporated in the Swift language design.

# A Third-party Future

I'm going to assume a basic understanding of [Futures](https://en.wikipedia.org/wiki/Futures_and_promises) and how they improve asynchronous programming. Consider the following example:

{% highlight swift %}
let string = fetchAnswer().map { ans in
    return ans * 2
}.map { timesTwo in
    return "Answer times two is " + timesTwo
}
// string is a Future<String, E>
{% endhighlight %}

The constant `string` is a Future string, it is a placeholder of a string value that depends on the outcome of the asynchronous operator `fetchAnswer` and the manipulations that follow. If the answer returned from the server is '21', the resulting string will be 'Answer times two is 42'.

While this is an improvement over the vanilla solution (i.e. completion handlers), it adds a lot of noise. For example, all changes are wrapped in `map` closures. We can do a little bit better, without having to change Swift itself. We can define an alternate version of the multiplication operator (`*`) that works on a Future int and a regular int and returns a Future int.

{% highlight swift %}
func *<E>(lhs: Future<Int, E>, rhs: Int) -> Future<Int, E> {
    return lhs.map { l in
        return l * rhs
    }
}
{% endhighlight %}

This would allow us to write the following:

{% highlight swift %}
let answerTimesTwo = fetchAnswer() * 2
// answerTimesTwo is a Future<String, Error>
{% endhighlight %}

This removes any visual overhead that Futures have and leaves us with a seemingly boring multiplication between two regular ints. The second part, where the int is appended to a string using string interpolation is not possible without changes to the language.

A few more examples of what would be feasible as a third-party Future library developer:

{% highlight swift %}
let str = stringAtUrl("http://example.com/helloworld")
// str is a Future<String, Error>
let lcStr = string.lowercaseString
// lcStr is a Future<String, Error>
let hasPr = "helloworld example".hasPrefix(lcString)
// hasPr is a Future<Bool, Error>
{% endhighlight %}

On line 3, `lowercaseString` is called on a Future string. This is a standard library function on the String type that can be redefined on all Futures that contain a string. On line 5, a Future string is passed as the parameter to `hasPrefix`. This is again a standard library function that usually takes a regular String. We can add a *future proof* version to the String type using an extension. The necessary additions that make the code sample from above work are:

{% highlight swift %}
extension Future where Value == String {
    var lowercaseString: Future<String, Value.Error> {
        return map { s in
            return s.lowercaseString
        }
    }
}

extension String {
    func hasPrefix<E>(prefix: Future<String, E>) -> Future<Bool, E> {
        return prefix.map { s in
            return self.hasPrefix(s)
        }
    }
}
{% endhighlight %}

While it would be possible to programmatically generate Future-compatible versions of all built-in operators and functions, I think we could go a lot further if the language and standard library would come with support for asynchronous values.

For the remainder of this post, I'd like to think about what built-in support for asynchronous values would look like. Needless to say, anything beyond this point is merely a thought experiment, a mental preparation for a pull request that I'd love to file once Swift is open sourced. **It's Swift fan fiction.**

# The Async Type

I think it would be interesting if Swift would support Futures similar to how Optionals are implemented. There's the `Optional` enum, but you hardly ever use it directly. At the same time, almost all functionality of the optional type would work without language support. I really like this approach and would love to see a powerful Future type in the standard library that is boosted by special treatment by the compiler.

Firstly we need a way to mark a type as asynchronous, like the question mark for optionals. Let's use the `async` keyword and allow it to be put in front of a type. If it is used in the return type of a function, the function becomes an asynchronous operation. Any functions called on an `async` type also become asynchronous operations.

Let's revisit the previous example:

{% highlight swift %}
func stringAtUrl(url: String) -> async String { 
    /* ... */ 
}

let str = stringAtUrl("http://example.com/helloworld")
let lcStr = string.lowercaseString
let hasPr = "helloworld example".hasPrefix(lcString)
{% endhighlight %}

The outcome of this code sample is `hasPr`, which is an `async Bool`. 

For a lot of functions, operators included, the compiler could just generate asynchronous variants on the fly. For other situations, there should be a way to turn an asynchronous type into a regular type:

{% highlight swift %}

let asyncStr = stringAtUrl("http://example.com/helloworld")

when let str = asyncStr else {
    throw StringLoadingFailed()
}

textLabel.setText(str)
{% endhighlight %}

The `when let` statement pauses the execution of the current scope until the asynchronous string value has been resolved (i.e. fetched from the network). If the operation failed, the `else` clause is executed. If successfull, the execution will continue with a regular String value in `str`. While the execution is paused, control is returned to the previous frame on the stack. The return type of the scope that contains the `when let` statement has to be marked `async`. If it does not have a return type, it has to be marked as `async Void`.

To check if an asynchronous value has already been resolved, optional*esque* syntax could be used: `if let str = asyncStr { ... }`, `asyncStr?.lowercaseString` as well as force unwrapping an asynchronous value using the exclamation mark, risking a runtime exception if the value is not yet there.

CLOSING PARAGRAPH NEEDED

<!-- If Swift wouldn't have optionals, third-party frameworks would be able to provide an implementation that is almost as powerful as the built-in Optional that we've come to know an love. Only `if let` statements, syntactic sugar in parameters and return types, and passing non-optional values as parameters to a function that expect their optional counterparts require special treatment by the compiler. -->

<!-- ENDING: There's the concept 'asynchronous values' and there's the class 'Future'. You'd be using both, but you often only see the first. You'd probably write 'let a: async Int' and not `let a: Future<Int, Error>`. The latter reveals the implementation of the 'asynchronous values' concept. The language can hide the implementation with syntactic sugar, enabling a great way to deal with the complexities of asynchronous development. -->

<!-- Anything could be asynchronous, so it should be possible to mark anything as asynchronous. To see what the right approach for this would be, we can turn to the Swift grammar. The grammar is described and - to some extent - explained in the last chapter of the [Swift Programming Language](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/) book.

[type rule](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Types.html#//apple_ref/doc/uid/TP40014097-CH31-ID445):

> *type* → *array-type \| dictionary-type \| function-type \| type-identifier \| tuple-type \| optional-type \| implicitly-unwrapped-optional-type \| protocol-composition-type \| metatype-type*

The *type* rule describes that a type can consist of one of the following: an array type (`[Int]`), a dictionary type (`[Int:String]`), function type (`Int -> String`) etc. A particularly interesting option is the `optional-value`, which refers to the following rule:

> *optional-type* → *type* **?**

An optional type is a type followed by a question mark. *type* on the right side of the arrow refers to the rule above, so what you put in front of the question mark can be anything that is a type: an array, a dictionary and even another optional type. Turning it around: you can make any type optional by putting a question mark after it. That is exactly what we want for the asynchronous values: mark any type as asynchronous by putting `async` in front of it. This is described by the following rule:

> *async-type* → **async** *type*

This means you can have asynchronous arrays, functions, optionals and even recursive asynchronous types. When we then add *async-type* as one of the options on the right side of the arrow of the *type* rule, we can use `async` in all the appropriate places. The following then becomes valid Swift: -->





<!-- This whole idea revolves around a new language keyword: `async`. It is a type qualifier, meaning that you can use it wherever you use a type. Add it to a type to indicate asynchronicity. Some examples:

- `func fetch(url: String) -> async NSData`   
	A function that performs an asynchronous operation that will return a `NSData` instance
- `let data: async NSData = fetch(url)`  
	A constant with the result of an asynchronous operation.

Up until some moment in time, the operation that `data` represents the result of has not yet been completed. Up until that moment in time, `data` is empty. To be able to do something with the data, we need to get the actual value from ‘inside’ the async container: we need to unwrap `data`. This is similar to how we work with optionals, except that it is not a question of ‘if’ but ‘when’:

{% highlight swift %}
func json() -> async [String:AnyObject] {
	let asyncData = fetch()
	
	when let data = asyncData {
		return parse(data)
	}
}
{% endhighlight %}

When the execution of the function arrives at the `when let` statement and the fetch operation is not yet completed, the `json()` function returns and execution at the callsite continues. When the `fetch` operation is completed, the `when let` body will be executed and the `async` dictionary returned from `json()` will be completed as well (triggering `when let` expressions that wanted to unwrap the `json()` result).

Using `when let` automatically requires the method to declare its resulting type as `async`, because it depends on the result of another asynchronous operation (i.e. `fetch()`). To prevent confusing code paths, `when let` must be the last expression in a function or must be nested the last expression inside another `when let` block.

Force unwrapping (`!`) and optional chaining (`?`) could work the same way as they do on optionals.

# A fluent interface to asynchronous values
Futures enable you to work with asynchronous operations (and their results) as if they were synchronous:

{% highlight swift %}
let string = fetchAnswer().map { ans in
	ans * 2
}.map { timesTwo in
	"Answer times two is \(timesTwo)"
}
{% endhighlight %}

This sample constructs the value of `string`, which is the result of an asynchronous operation  `fetchAnswer()`, times two and formatted in a string. Futures hide the complexity that you have to deal with around asynchronous operations. The exact same code would be valid if `fetchAnswer()` were to return an optional.

It is however not as straight forward as dealing with regular synchronous values. All operations are performed inside a map closure, adding significant visual noise. What if all functions (and operators) accepted asynchronous versions of the parameter types they are defined with? That would make it possible to write the following:

{% highlight swift %}
let timesTwo: async Int = fetchAnswer() * 2
let string: async String = "Answer times two is \(timesTwo)
{% endhighlight %}

Again: The explicit type declarations were added for clarity, but should be unnecessary in real code.

The times operator (‘\*’) used in the code above is a function that takes two ints and returns an int. Because we pass it an `async Int`, its return type is automatically turned into an `async` version of the defined type.

From this, it is just a small step to allow calling functions of async types on the type they are wrapping:

{% highlight swift %}
let timesTwo: async Int = fetchAnswer().times(2)
{% endhighlight %}

(assuming there is a `times` function defined on `Int`)

{% highlight swift %}
let context: AnyObject?
let context: Optional<AnyObject>


let data: async NSData
let data: Async<NSData>


let data = session.fetch("http://www.example.org/birds.json")
let json: async [Dictionary<String, AnyObject>] = parse(data)
let birds = json.map(Bird.init)
let images = birds.map { session.fetchImage($0.imageUrl) }
let birdsAndImages: async [(Bird, UIImage)] = zip(birds, images)

when let birdsAndImages = birdsAndImages {
		tableView.reload()
}
{% endhighlight %} -->

<!-- So what does it mean for something to be a *first-class citizen*? Future is a class and a class is a first-class citizen. That makes a Future a first-class citizen *by proxy*. Just like Optional, which is an enum and thus a first-class citizen. The language does however provide some syntactic sugar for working with optionals that seem to make them first*er*-class than Futures. -->