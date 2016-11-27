---
layout: post
title: "Parsing JSON"
---

Inspired by the recent Swift Talk videos on parsing, I decided to apply the things I learned to an every-day task: parsing JSON. Since there's already a multiplitude of libraries that help you going from AnyObject to your models, I decided to go one level deeper and build a replacement for NSJSONSerialization, using parser combinators. A few things to note:

- there is (probably) no good reason ever not to use NSJSONSerialization
- building an alternative it's still a good exercise
- but I won't attempt to implement the complete JSON spec

We're going to build a parser that takes a JSON string as input, and returns an object representation of said JSON. I think we can do better than just returning AnyObject so we'll return an instance of JSONValue:

```swift
enum JSONValue {
    case Array([JSONValue])
    case Object([String:JSONValue])
    case String(String)
    case Integer(Int)
    case Boolean(Bool)
    case Null
}
```

Since parser combinator code often closely resembles the structure of the parsed language, we'll start out by defining the grammar of the JSON subset we'll be covering in this post:
 
> *array* → `[` (*value* `,`)<sub>many</sub> *value* `]`
>
> *object* → `{` (*pair* `,`)<sub>many</sub> *pair* `}`
> 
> *pair* → *string* `:` value
>
> *string* → `"` Any character except "<sub>many</sub> `"`
>
> *integer* → Digit 0 through 9
>
> *boolean* → `true` \| `false`
>
> *null* → `null`
>
> *value* → *array* \| *object* \| *string* \| *integer* \| *boolean* \| *null*

(This notation is similar to the one used by Apple to describe the [Swift grammar](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AboutTheLanguageReference.html#//apple_ref/doc/uid/TP40014097-CH29-ID345))

A naive implementation of this grammar would yield a stack overflow as soon as we'd try to parse our first piece of JSON. There are circular references in the grammar, e.g. an array contains values which could be an array which in turn contains values which... you get the point.

We can solve this by deferring the creation of some of the parsers only when other options fail. So instead of creating the array parser when parsing a value, we'll only create the array parser if the value is none of the other value types. With that in place, we'd only trigger a stack overflow if the JSON we're trying to parse had an incredible depth. This is acceptable.
