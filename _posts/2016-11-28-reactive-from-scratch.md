---
layout: post
title: "Reactive programming from scratch"
---

Let's write a minimal Reactive program. From scratch. Let's reinvent a very tiny wheel, for the purpose of really understanding. When there're so few lines of code, there's not much to be explained. It's all there, at a glance.

<!-- The more lines you write, the bigger the chance you introduce something that is perceived as magic. And while there is no such thing as magic in programming, a collective of code could keep up the appearance of being magic. -->

If you understand the code from this post today, you'll have an easier job understanding RxSwift tomorrow. So let's get started.

At the core of Reactive programming is a concept called an Observable. Think of it as an asynchronous array. It has a beginning and an end, with elements in between. We could define a simple Observable containing the numbers 1, 2 and 3 as follows:

```swift
let o = Observable<Int> { observer in
    observer(.next(1))
    observer(.next(2))
    observer(.next(3))
    observer(.completed)
}
```

To work with the contents of an Observable, you need to subscribe to it. When subscribing, you pass in a closure (the Observer) that is called for each event:

```swift
o.subscribe { e in
    // this closure is invoked four times
    // e is .next(1), .next(2), .next(3), .completed
}
```

This is very similar to iterating over an array, especially if you use an explicit Iterator. Instead of calling `next()` until it returns `nil`, you receive next events until it completes. (For the theoretically inclined: Iterators and Observers are mathematical duals.)

Now let's look at the 22 lines that are needed to make the examples shown so far compile and work as advertised:

```swift
enum Event<Element> {
    case next(Element)
    case error(Error)
    case completed
}

class Observable<Element> {
    
    typealias SubscribeHandler = (@escaping Observer) -> ()
    typealias Observer = (Event<Element>) -> ()
    
    let subscribeHandler: SubscribeHandler
    
    init(subscribeHandler: @escaping SubscribeHandler) {
        self.subscribeHandler = subscribeHandler
    }
    
    func subscribe(observer: @escaping Observer) {
        subscribeHandler(observer)
    }
    
}
```

An Observable is initialized with a description of what its contents looks like, _in terms of_ what an Observer will receive. Whenever an Observer subscribes, it will start receiving the contents. The speed and timing with which events are emitted is up to the Observable. The Observable contains any number of Events: zero or more nexts, followed by either an error or completed.

With this simple API, the same we used to define a simple number sequence, we can define an Observable that represents a network request. We can create an extension on URLSession with a method that wraps the default data task API:

```swift
extension URLSession {
    func response(url: URL) -> Observable<(Data, URLResponse)> {
        return Observable { observer in
            self.dataTask(with: url) { data, response, error in
                if let data = data, let response = response {
                    observer(.next(data, response))
                    observer(.completed)
                } else {
                    observer(.error(error!))
                }
            }.resume()
        }
    }
}

session.response(url: "http://www.thomvis.nl").subscribe { e in
    // if the request succeeds, e is .next(data, response) & .completed
    // if the request fails, e is .error(httpError)
}
```

When an Observer subscribes to the returned Observable, the request is initiated. Once the request finished, the data and response are emitted to the Observer. If the request failed, the Observer will receive an error. Note that when a second Observer would subscribe to the same Observable, the request would be executed a second time.

The examples so far show that Observables can be applied in various use cases, ranging from simple sequences, like the one at the top of this post, all the way up to complex and asynchronous operations, like network requests. They can also simplify working with UI events or continuous model changes.

There are numerous ways to extend the simple Observable type we created above to make it more useful. Let's first look at a way to simplify the conversion from a plain array to an Observable:

```swift
extension Observable {
    convenience init(elements: [Element]) {
        self.init { observer in
            for e in elements {
                observer(.next(e))
            }
            observer(.completed)
        }
    }
}

Observable(elements: [2, 4, 6, 10]).subscribe { e in
    // e is .next(2), .next(4), .next(6), .next(10), .completed
}
```

A plain array, e.g. with URLs of images to fetch, is a common starting point for an Observable.

We can start building bigger Reactive expressions by introducing ways to compose Observables. Let's implement `map`. You might have used it to turn one optional type to another optional type or to create a new array with a new value for each element in another array. For Observables it does exactly the same: it takes a function that describes how a single element is transformed and returns an Observable that contains a transformed element for each element in the original Observable.

```swift
extension Observable {
    func map<T>(f: @escaping (Element) -> T) -> Observable<T> {
        return Observable<T> { observer in
            self.subscribe { event in
                switch event {
                case .next(let e):
                    observer(.next(f(e)))
                case .error(let error):
                    observer(.error(error))
                case .completed:
                    observer(.completed)
                }
            }
        }
    }
}

let o3 = Observable(elements: [1, 2, 3]).map { $0 % 2 == 0 }

o3.subscribe { e in
    // e is .next(false), .next(true), .next(false), .completed
}
```

When an Observer subscribes to the returned Observable, it in turn subscribes to the original Observable. The error and completed events are passed on unchanged, but the nexts are created with values returned from `f`.

In the example above, an Observable containing integers is _mapped_ to an Observable with booleans. True is emitted for each even number and false otherwise.

For the finale of this post, we will look at how to perform any number of network requests (two in this case) and create a single Observable containing their responses.

```swift
let urls = ["http://www.thomvis.nl", "http://www.thomasvisser.me"]
let responses = Observable(elements: urls).map { 
    session.response(url: URL(string: $0)!) 
}
// responses is Observable<Observable<(Data, URLResponse)>>
```

Starting with an array of URLs, we wrap them in an Observable and map each URL to an Observable that represents the response of the request to that URL. The result is an Observable consisting of Observables consisting of data & response tuples.

This additional layer of Observables complicates working with the actual URL responses. Similarly to how this works with two dimensional arrays and nested optionals, we can flatten this Observable. The result is an Observable that contains all elements from the inner Observables.

```swift
func flatten<E>(_ o: Observable<Observable<E>>) -> Observable<E> {
    return Observable { observer in
        var subscriptions = 0
        var terminated = false
        o.subscribe { e in
            switch e {
            case .next(let innerObservable):
                subscriptions += 1
                innerObservable.subscribe { e in
                    switch e {
                    case .next(let e):
                        observer(.next(e))
                    case .error(let e):
                        observer(.error(e))
                    case .completed:
                        subscriptions -= 1
                        
                        if (terminated && subscriptions == 0) {
                            observer(.completed)
                        }
                    }
                }
            case .error(let e):
                observer(.error(e))
            case .completed:
                terminated = true
                
                if (terminated && subscriptions == 0) {
                    observer(.completed)
                }
                break
            }
        }
    }
}
```

Inside `flatten`, we subscribe to each inner Observable as it is emitted by the outer Observable. We emit each next event of the inner Observables to the Observer. The resulting Observable is completed once every inner Observable _and_ the outer Observable have completed.

Putting everything together, here's the final example of how to use Reactive programming for fetching multiple URLs and dealing with their responses as they come in. It doesn't look like much, and that's a good thing.

```swift
flatten(Observable(elements: [
    "http://www.thomvis.nl", 
    "http://www.thomasvisser.me"
]).map { 
    session.response(url: URL(string: $0)!) 
}).subscribe { e in
    // e is two .next events with the responses from the requested URLs
    // followed by a .completed
}
```

The code in this post is not intended for use serious projects. A full implementation of Reactive programming concepts has to face challenges related to threading, memory management and more. I just ignored them. 

If you want to use Reactive programming in your projects, have a look at [RxSwift](https://github.com/ReactiveX/RxSwift), which I use, or [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa), which I have heard good things about.

