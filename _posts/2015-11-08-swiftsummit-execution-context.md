---
layout: post
title: "Deleted scenes from my Swift Summit talk"
---

Last week, I gave a talk at [Swift Summit](https://www.swiftsummit.com) in San Francisco wherein I live coded a minimalistic Futures implementation based on [BrightFutures](https://github.com/Thomvis/BrightFutures). It was my first conference talk, but I think it went well and it was a lot of fun to do.

While preparing for the talk it became clear that the biggest challenge would be to fit my story in the assigned 18-minute slot. I ended up cutting half of the content. The result was a to the point, fast-paced, concise presentation; it was probably for the best.

Of the content that I had to remove from the talk, there's one particular part that I regret removing the most: the part that showed how to elegantly specify the thread where a callback should be called on. This is important because some stuff can only be done on the main thread, while other logic should be kept off of the main thread in order to keep your app responsive.

The following video was recorded during a practice run of my talk for my collegues at [Highstreet](http://highstreetapp.com) & [Touchwonders](http://www.touchwonders.com). The remainder of this post is a loose transcription of that video, in case you prefer that.

<iframe width="700" height="393" style="margin-top:1em;margin-bottom:1em;" src="https://www.youtube.com/embed/XOnyQ9qzz38" frameborder="0" allowfullscreen></iframe>

The video starts at the point in my presentation where I've created a basic implementation of Async: a simple alternative for completion handlers. (It is similar to the [one](https://github.com/Thomvis/SFSwiftSummit2015/blob/master/SwiftSummit/Async.swift#L21) I used in the final version of my talk.) It represents the result of an asynchronous operation and allows for the registration of completion callbacks that will be invoked with the result of the operation when it has completed.

In the sample app, I have an asynchronous operation that fetches a list of birds [json file](https://github.com/Thomvis/SFSwiftSummit2015/blob/master/naming.json) and their images from the internet. Once the operation is complete, the data is stored in the view controller and `reloadData` is called on the table view:

{% highlight swift %}
dataSource.getBirds().onComplete { birds in
    self.birds = birds
    self.tableView.reloadData()
}
{% endhighlight %}

Running the app however reveals a bug: the contents of the table view only appears after the user tries to scroll. These kind of things tend to happen when you use UIKit from a background thread and in this particular case, that is exactly what is happening. The operation, `dataSource.getBirds()`, uses data returned from `NSURLSessionDataTask`s and eventually completes the `Async`, which calls the `onComplete` closure, without switching threads. `reloadData` is therefore called on the thread that `NSURLSession` called back on, which is not the main thread.

To solve this issue, we need a way to specify *where* the callback should be invoked on, i.e. a specific thread or queue, when we register the callback. At the same time, we gain a huge advantage over the traditional completion block based API because it will enable the caller to specify the thread the callback should be invoked on.

In order to properly abstract the details of threads and queues, we need execution contexts, a term I learned from [Scala](http://www.scala-lang.org/files/archive/nightly/docs/library/index.html#scala.concurrent.ExecutionContext). An execution context can execute logic, on a specific thread (pool) or queue. In Swift, we can define an execution context as a function that takes a closure, which is the task to execute:

{% highlight swift %}
typealias ExecutionContext = (() -> Void) -> Void
{% endhighlight %}

With that in place we can start creating useful contexts. By creating an extension on `dispatch_queue_t` we can give every GCD queue an execution context:

{% highlight swift %}
extension dispatch_queue_t {
    var context: ExecutionContext {
        return { task in
            dispatch_async(self, task)
        }
    }
}
{% endhighlight %}

One context that will be particularly useful when building apps is one that performs tasks on the main queue. This is now as easy as getting `context` from the main queue:

{% highlight swift %}
let MainExecutionContext = dispatch_get_main_queue().context
{% endhighlight %}

Another useful context just immediately executes a task without switching threads:

{% highlight swift %}
let ImmediateExecutionContext: ExecutionContext = { $0() }
{% endhighlight %}

We can now update `onComplete` to take an execution context as the first parameter. By default it makes sense to use the immediate execution context. In order to remember which callback should be invoked on which context we can simply create a new callback that wraps the given callback and executes it on the given context. That wrapped callback is then either executed immediately (if the operation is already completed) or appended to the array with pending callbacks:

{% highlight swift %}
func onComplete(context: ExecutionContext = ImmediateExecutionContext, callback: Result -> Void) -> Self {
    let wrappedCallback: Callback = { res in
        context {
            callback(res)
        }
    }
    
    if let result = result {
        wrappedCallback(result)
    } else {
        callbacks.append(wrappedCallback)
    }
    
    return self
}
{% endhighlight %}

Now as the final step, to fix the threading issue with the `reloadData`, we can pass the `MainExecutionContext` when registering the callback in the view controller:

{% highlight swift %}
dataSource.getBirds().onComplete(MainExecutionContext) { birds in
    self.birds = birds
    self.tableView.reloadData()
}
{% endhighlight %}

This fixes the threading issue in what I think is a nice way: it is simple, composable and *powered by Swift*.

This is exactly how threading is implemented in BrightFutures. If you like it, [give BrightFutures a try](https://github.com/Thomvis/BrightFutures).


