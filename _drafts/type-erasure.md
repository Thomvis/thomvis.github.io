---
layout: post
title: "A tale of two type erasures"
---

At university, I learned about type erasure. It meant that generics in Java did not exist on runtime.

When the community started learning Swift, I learned about another thing called type erasure.

Those two type erasures turned out to be one thing.


I got to know type erasure as a bad thing. At the time, I was learning to program in Java and type erasure was the reason why we .

[A lot](http://robnapier.net/erasure) [has been written](https://realm.io/news/type-erased-wrappers-in-swift/) about type-erasure in Swift. From reading those posts, I got the feeling that the technique was primarily presented as a feature, something cool you can do with Swift. While this is true for some cases, I got to know type erasure first as a limitation and now as a workaround for limitations of Swift. In this post, I'd like to provide some additional context to the type erasure technique both as something that is useful and necessary evil.

I first heared about 'type erasure' was in the context of Java. There I got to know it as a bad thing. Every time that I came up with some cool generic data structure, it turned out not to work and that was because of type erasure. Type erasure meant that all the rich generic type information would be gone on runtime. 

Quick example: a generic `List<E>` type would not be able to know what `E` is on runtime, check if an object is of type `E`, create a new instance of `E`, etc. It even has its own [section](https://en.wikipedia.org/wiki/Generics_in_Java#Problems_with_type_erasure) in the Wikipedia article about generics in Java.

All the things that I described above that you can't do with generic types in Java, you can do in Swift. Swift preserves all type information for runtime. This is awesome! However, there are a few places in the language where the type information is more detailed than what you can describe using the current valid syntax.

As an example, let's create a very simple protocol with an associated type:

{% highlight swift %}
protocol BoxType {
    typealias BoxedType
    
    var value: BoxedType { get }
}
{% endhighlight %}

Now what we'd like to do is have an array of instances that conform to the BoxType protocol. We could try something like this:

{% highlight swift %}
let boxes = [BoxType]()
{% endhighlight %}

You'll probably already know that this doesn't work and indeed, Xcode gives the following error:

`Protocol 'BoxType' can only be used as a generic constraint because it has Self or associated type requirements`

There is no way to specify what type the associated `BoxedType` is and Swift is not able to create an array without precisely knowing the type. I think this is a limitation of the language and it is hopefully one that will be addressed in a future version. My take on it would be to introduce syntax like this:

{% highlight swift %}
let boxes = [BoxType where BoxedType == String]()
{% endhighlight %}

But until that is the case, the solution would be to use type erasure. This entails creating an `AnyBox<E>` like this:

{% highlight swift %}
class AnyBox<E>: BoxType {
    
    let _value: () -> E
    
    init<B: BoxType where B.BoxedType == E>(box: B) {
        _value = { box.value }
    }
    
    var value: E { _value() }
    
}
{% endhighlight %}

Now we can create an array of boxes:

{% highlight swift %}
let boxes2 = [AnyBox<String>]()
{% endhighlight %}

CLOSING PARAGRAPH NEEDED