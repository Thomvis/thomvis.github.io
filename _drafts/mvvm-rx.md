---
layout: post
title: "Reactive MVVM"
---

There are many ways to implement MVVM or to put RxSwift to use, let alone combine the two. But a fine combination they make, and the details of an implementation of Reactive MVVM determine how testable your code is, how flexible or concise. That's why it is interesting to look at a specific implementation, as we'll do in this post today.

The presented approach is _just a way_ to combine RxSwift and MVVM in the context of iOS apps. It has served us well over the last year at [Highstreet](http://www.highstreetapp.com). I hope you'll find it equally helpful.

To help appreciate the benefits of this implementation, we first need to look at the challenges we had before arriving at what now seems so logical. Let's begin with an overview of typical view model usage:

```swift
// 1. initialize the view model with dependencies & model
let vm = ViewModel(apiClient: apiClient, database: db, model: model)

// 2. provide the view model with inputs from the view
vm.setInputs(button.rx.tap, usernameField.rx.text)

// 3. subscribe the view to the outputs from the view model
vm.buttonEnabled().subscribe(onNext: {
    button.setEnabled($0)
}).addDisposableTo(bag)
```

You probably won't find, or want to find, these lines of code so close together in a serious code base. It presumes a single context where the app's infrastructure, models and view are within reach. Such a context is likely to be tightly coupled to other parts of the app and will prove to be hard to test.

But as you move these lines apart, to their respective correct location within your app's architecture, a new issue arises. There's an implicit requirement on the order in which step 2 and 3 are executed. The reason why becomes apparent as we look at a typical implementation of the view model:

```swift
class ViewModel {
    var buttonTap: Observable<Void>!
    var usernameText: Observable<String>!

    // initializer omitted

    func setInputs(tap: Observable<Void>, username: Observable<Void>) {
        buttonTap = tap
        usernameText = username
    }

    func buttonEnabled() -> Observable<Bool> {
        return usernameText.map { $0 != nil }
    }
}
```

If the view subscribes to `buttonEnabled()` before `setInputs` is called, the app will crash. Making `usernameText` a regular Optional is not a solution, because then we have to handle the nil case in `buttonEnabled()` (e.g. by returning an empty Observable), which can only lead to unexpected behavior.

Another solution is to make `usernameText` a [Subject](http://reactivex.io/documentation/subject.html). The Subject can be created when the view model is initialized and after that, the order in which `setInputs` and `buttonEnabled` are called doesn't really matter. The downside is however that you'll know have to manage a subscription on `username` in `setInputs` and deal with additional state that comes with a Subject. The latter being the reason why you might have heard that [Subjects should be avoided](http://introtorx.com/Content/v1.0.10621.0/18_UsageGuidelines.html). They definately have their use, but I think we can do better here.

As we've been fiddling with Optionals and Subjects, we've been avoiding the real issue here: we're dealing with an object that has an inherently uncomfortable, transient state. The use of multiple Optionals at a single level are a symptom of this. A solution can be to look one level up.

We can get rid of the optionals by refactoring ViewModel to have an initializer that receives all required dependencies: infrastructure, model and view inputs:

```swift
class ViewModel {
    let buttonTap: Observable<Void>
    let usernameText: Observable<String>

    init(apiClient: APIClient, database: DB, model: Model, tap: Observable<Void>, username: Observable<String>) {
        buttonTap = tap
        usernameText = username
    }

    func buttonEnabled() -> Observable<Bool> {
        return usernameText.map { $0 != nil }
    }
}
```

This has greatly simplified the inner workings of the ViewModel: all properties are constant, `setInputs` is gone and there's no (explicitly unwrapped) Optional in sight.

But as we return to the code where the view model is initialized, we notice things took a turn for the worse. The three steps have been replaced by two (as step one and two are now combined) and a single context, a single line even, in which both the app's infrastructure and the view are accessible is now inevitable.

```swift
// 1. initialize the view model with dependencies, model & view
let vm = ViewModel(apiClient: apiClient, database: database, model: model, tap: button.rx.tap, username usernameField.rx.text)
```

We can resolve this by reintroducing a way to represent the two states the view model can be in: the initial state where it is detached from the view and the state where it is all wired up.

The power of the solution that we're looking for will come, in part, from the fact that it doesn't care (i.e. make no assuptions) on the inner workings of the view model. This does require us to briefly formalize what a view model looks like from the outside and how it can be initialized:

```swift
protocol ViewModelProtocol {
    associatedtype Dependencies
    associatedtype Model
    associatedtype Bindings

    init(dependencies: Dependencies, model: Model, bindings: Bindings)
}
```

The dependencies are parts of the app's infrastructure that are passed to each instance of that view model. The model contains data that is specific to that instance of the view model. The bindings define the ways the view model can observe the view, for as long as it is attached to that view.

Making our view model conform to the protocol is quite straight forward:

```swift
class ViewModel: ViewModelProtocol {
 
    // properties & buttonEnabled() omitted

    init(dependencies: Dependencies, model: Model, bindings: Bindings) {
        buttonTap = bindings.tap
        usernameText = bindings.username
    }

    struct Dependencies {
        let apiClient: APIClient
        let database: DB
    }

    typealias Model = () // this VM has no model

    struct Bindings {
        let tap: Observable<Void>
        let username: Observable<String>
    }
}
```

We can now create a type that wraps around a view model to represent the two states it can be in:

```swift
enum Attachable<VM: ViewModelProtocol> {
    
    case detached(VM.Dependencies, VM.Model)
    case attached(VM.Dependencies, VM.Model, VM)
    
    mutating func bind(_ bindings: VM.Bindings) -> VM {
        switch self {
        case let .detached(deps, model), let .attached(deps, model, _):
            let vm = VM(
                dependencies: deps,
                model: model,
                bindings: bindings
            )
            self = .attached(deps, model, vm)
            return vm
        }
    }
    
}
```

An `Attachable<ViewModel>` ("attachable view model") can be created in detached state by providing its dependencies and its model. This is typically done outside of the view, in a coordinator, parent view model or router. In detached state, the view model is passed to the view (controller), where it is bound to the view. The result is an attached view model, giving access to the actual view model instance. The view model instance is conveniently returned from the `bind` method.

These steps are consolidated in the following example:

```swift
// 1. initialize the view model with dependencies & model
let avm: Attachable<ViewModel> = .detached(dependencies, ())

// 2. provide the view model with inputs from the view
let vm = avm.bind(view.bindings)

// 3. subscribe the view to the outputs from the view model
vm.buttonEnabled().subscribe(onNext: {
    button.setEnabled($0)
}).addDisposableTo(bag)
```

At first glance, this last example appears to be very similar to the situation we started this post with. We have however achieved two important things: the inner workings of the view model have been simplified and the Attachable type guarantees correct usage of the view model from the outside. 

The presented approach is unobtrusive, it consists of just one protocol and an enum. It encourages flexibility by using composition, instead of requiring a superclass. It can also be extended to support more complex use cases, for example repeated attaching/detaching in the context of a collection view or using parts of the view model's logic before it has been attached through protocol extensions on Attachable.

It is by no means the final say on Reactive MVVM. But an encouragement to continue exploring within this paradigm it is.
