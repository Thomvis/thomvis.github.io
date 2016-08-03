---
layout: post
title: "Presenting Asynchronous Content with RxSwift"
---

The topic of this post is weirdly specific for a first post on [RxSwift](https://github.com/ReactiveX/RxSwift), but it's a good a start as any.

As of late, we've started writing new features for [Highstreet](http://www.highstreetapp.com) in Swift. Coming back to the iOS side of things after spending 6 months on building the Android app with RxJava, I found it hard to write new code without the use of a form of [Rx](http://reactivex.io). We're using MVVM, which works really well combined with a reactive approach. It was not before long until I added RxSwift to the Podfile.

Let me set the scene: imagine a view that displays content that has to be loaded from the network. We'll start loading the content in `viewWillAppear` and present it as soon as it's there. The result? A view that shows the data as soon as possible, but might flicker on appearance depending on network latency.

If we want to do better, we could try to do something like this: if the data is ready (almost) immediately, we show the data right away. If the data is not ready, we show a loading indicator. And when we show the loading indicator, we want to show it for at least a second before we display any data, to make sure that doesn't cause a flicker either.

A solution without RxSwift would typically involve a timer, some instance variables to store the state and logic that is spread out across at least two methods. I think we can do a lot better with Rx:

```swift
enum DataState<E> {
    case Loading
    case Done(E)
}

let source: Observable<String> = // an asynchronous String

func data() -> Observable<DataState<String>> {
    let data = source.map { DataState.Done($0) }
    let loading = Observable<Int>
		.timer(0.05, scheduler: MainScheduler.instance)
		.map { _ in DataState<String>.Loading }
    let delayedData = data
    	.delaySubscription(1.0, scheduler: MainScheduler.instance)
    
    let loadingThenData = loading.concat(delayedData)
    
    return [data, loadingThenData].amb()
}
```

The asynchronous content is represented by the `source` Observable in this example. Let's assume it will return one string after an undetermined amount of time. The `data()` function returns an Observable that will emit one of the following:

- a `.Done` item if it arrives within 0.05 seconds
- a `.Loading` item after 0.05 seconds, followed by a `.Done` item after at least 1 second

In the code, you can see that I'm creating two Observables, one for each of the scenarios above. `data` is a direct mapping of the source, immediately emitting `.Done` items for every item the source emits. `loadingThenData` is the result of two Observables that are being concatenated: the first one is an Observable that emits `.Loading` after 0.05 seconds, the second is identical to `data`, but with a subscription delay of 1 second. This ensures there will always be at least one second between `.Loading` and `.Data`.

The `amb` operator subscribes to both Observables and returns the items from the first one that will emit an item (i.e. fastest wins, [more info](http://reactivex.io/documentation/operators/amb.html)). If the data Observable emits `.Data` within 0.05 seconds, it subscribes to `data`. If not, it subscribes to `loadingThenData`.

There's one caveat: the code above assumes that `source` has shared side-effects. One subscription to the Observable returned from `data()` can result in two subscriptions on `source`. Without the appropriate measures, this could lead to duplicate network requests. We can solve it using `share`, `shareReplay` or similar operators, but that would be a topic for a whole different post.

The takeaway of this post, I hope, is that RxSwift can help you write an elegant solution to a problem that otherwise would have consisted out of several moving parts. All logic is contained to this one function, all state is tied to the lifecycle of a subscription.

If only it didn't take a few weeks until I finally understood what a subscription was ;)
