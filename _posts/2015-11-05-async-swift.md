---
layout: post
title: "Getting ahead of myself: how built-in Futures could look like in Swift"
---

While I was preparing for my talk at SwiftSummit, I got a last minute idea on how Swift could incorporate Futures as a core language feature. This was on the morning of the second day of the conference, the day that I was speaking. The day before, I was taking part in a panel discussion on what will happen when Swift becomes Open Source. My brain must have mixed up the topic of the panel and the topic of my own talk and this little idea emerged. Too late for the presentation, I made some quick notes that I expanded on the flight back home. This is the expanded notes.

It’s a thought experiment, science fiction, a mental preparation for a pull request on the language & the standard library that I’d love to file once Swift is open sourced. (Help wanted!) I am not a programming language designer; the following might make sense to a varying degree.

This whole idea revolves around a new language keyword: `async`. It is a type qualifier, meaning that you can use it wherever you use a type. Add it to a type to indicate asynchronicity. Some examples:

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

	let string = fetchAnswer().map { ans in
		ans * 2
	}.map { timesTwo in
		"Answer times two is \(timesTwo)"
	}

This sample constructs the value of `string`, which is the result of an asynchronous operation  `fetchAnswer()`, times two and formatted in a string. Futures hide the complexity that you have to deal with around asynchronous operations. The exact same code would be valid if `fetchAnswer()` were to return an optional.

It is however not as straight forward as dealing with regular synchronous values. All operations are performed inside a map closure, adding significant visual noise. What if all functions (and operators) accepted asynchronous versions of the parameter types they are defined with? That would make it possible to write the following:

	let timesTwo: async Int = fetchAnswer() * 2
	let string: async String = "Answer times two is \(timesTwo)

Again: The explicit type declarations were added for clarity, but should be unnecessary in real code.

The times operator (‘\*’) used in the code above is a function that takes two ints and returns an int. Because we pass it an `async Int`, its return type is automatically turned into an `async` version of the defined type.

From this, it is just a small step to allow calling functions of async types on the type they are wrapping:

	let timesTwo: async Int = fetchAnswer().times(2)

(assuming there is a `times` function defined on `Int`)

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