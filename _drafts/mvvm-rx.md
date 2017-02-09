---
layout: post
title: "Reactive MVVM"
---

The MVVM architecture and the Reactive Programming paradigm can be combined to form a very pleasing approach to iOS app development. However, there's no clear-cut way of incorporating the two, and the details of the implementation determine how testable, flexible or concise your code is. That's why it's interesting to look at a specific implementation, as we'll do in this post today.

The presented approach is _just a way_ to combine RxSwift and MVVM in the context of iOS apps. It has served us well over the last year at [Highstreet](http://www.highstreetapp.com). I hope you'll find it equally helpful.

To help appreciate the benefits of this implementation, we first need to look at the challenges we had before arriving at what now seems so logical. Let's begin with an overview of typical view model usage:

```swift
// 1. initialize the view model with the model
let vm = ViewModel(model: model)

// 2. provide the view model with inputs from the view
vm.setInputs(button.rx.tap, usernameField.rx.text)

// 3. subscribe the view to the outputs from the view model
vm.buttonEnabled().subscribe(onNext: {
    button.setEnabled($0)
}).addDisposableTo(bag)
```

You probably won't find, or want to find, these lines of code so close together in a serious code base. It presumes a single context where the model, which can contain parts of your app's infrastructure, and view are within reach. Such a context is likely to be tightly coupled to other parts of the app and will prove to be harder to test.

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

If the view invokes `buttonEnabled` before `setInputs` is called, the app will crash. Making `usernameText` a regular optional is not a solution, because then we have to handle the nil case in `buttonEnabled()`, e.g. by returning an empty Observable:

```swift
func buttonEnabled() -> Observable<Bool> {
    return usernameText?.map { $0 != nil } ?? Observable.empty()
}
```

This leads to unexpected behavior for observers that subscribe to `buttonEnabled()` before `setInputs` is called. They have in fact subscribed to an empty observable and will never receive any items, even after `setInputs` has been called.

Another solution is to make `usernameText` a [Subject](http://reactivex.io/documentation/subject.html). The subject can be created when the view model is initialized and after that, the order in which `setInputs` and `buttonEnabled` are called doesn't really matter. However, you'll now have to manage a subscription on `username` in `setInputs` and deal with additional state that comes with a subject.

If you're new to Reactive programming, it might help to think of a subject as a mutable variable. This would make `usernameText` mutable, while it should be strictly read-only as far as the view model is concerned. This is the reason why you might have heard that [subjects should be avoided](http://introtorx.com/Content/v1.0.10621.0/18_UsageGuidelines.html). They definitely have their use, but I think we can do better without them here.

As we've been fiddling with optionals and subjects, we've been avoiding the real issue here: we're dealing with an object that has an inherently uncomfortable, transient state. The use of multiple optionals at a single level are a symptom of this. A solution can be to look one level up.

We can get rid of the optionals by refactoring ViewModel to have an initializer that receives all required dependencies: the model and view inputs.

```swift
class ViewModel {
    let buttonTap: Observable<Void>
    let usernameText: Observable<String>

    init(model: Model, tap: Observable<Void>, 
            username: Observable<String>) {
        buttonTap = tap
        usernameText = username
    }

    func buttonEnabled() -> Observable<Bool> {
        return usernameText.map { $0 != nil }
    }
}
```

This has greatly simplified the inner workings of the ViewModel: all properties are constant, `setInputs` is gone and there's no (explicitly unwrapped) optional in sight.

As we return to the code where the view model is initialized, we notice things took a turn for the worse. The three steps from before have been replaced by two (as step one and two are now combined) and a single context, a single line even, in which both model and view have to be accessible is now inevitable:

```swift
// 1. initialize the view model with the model & view bindings
let vm = ViewModel(model: model, tap: button.rx.tap, username: usernameField.rx.text)
```

Ideally, we want to initialize the view model outside of the view (controller), but we now need the view to initialize the view model first. We can solve this chicken and egg situation by reintroducing a way to represent the two states of the view model: the initial state where it's detached from the view and the state where it's all wired up.

The power of the solution that we're looking for will come, in part, from the fact that it doesn't care (i.e. makes no assuptions) about the inner workings of the view model. This does require us to briefly formalize what a view model looks like from the outside and how it can be initialized:

```swift
protocol ViewModelProtocol {
    associatedtype Model
    associatedtype Bindings

    init(model: Model, bindings: Bindings)
}
```

The model contains data, or ways to retrieve data, needed by the view model. The bindings define the ways the view model can observe the view, for as long as it's attached to that view.

Making our view model conform to the protocol is straightforward:

```swift
class ViewModel: ViewModelProtocol {
 
    // properties & buttonEnabled() omitted

    init(model: Model, bindings: Bindings) {
        buttonTap = bindings.tap
        usernameText = bindings.username
    }

    struct Model {
        let apiSession: Session
    }

    struct Bindings {
        let tap: Observable<Void>
        let username: Observable<String>
    }
}
```

We can now create a type that wraps around a view model to represent the two states it can be in:

```swift
enum Attachable<VM: ViewModelProtocol> {
    
    case detached(VM.Model)
    case attached(VM.Model, VM)
    
    mutating func bind(_ bindings: VM.Bindings) -> VM {
        switch self {
        case let .detached(model), let .attached(model, _):
            let vm = VM(model: model, bindings: bindings)
            self = .attached(model, vm)
            return vm
        }
    }
    
}
```

An `Attachable<ViewModel>` ("attachable view model") can be created in detached state by providing just the model. This is typically done outside of the view: in a coordinator, parent view model, router, etc. In detached state, the view model is passed to the view (controller), where it's bound to the view. The result is an attached view model, containing the actual view model instance. The view model instance is conveniently returned from the `bind` method.

These steps are consolidated in the following example:

```swift
// 1. initialize the view model with the model
let avm: Attachable<ViewModel> = .detached(model)

// 2. attach the view model to the view
let vm = avm.bind(view.bindings)

// 3. subscribe the view to the outputs from the view model
vm.buttonEnabled().subscribe(onNext: {
    button.setEnabled($0)
}).addDisposableTo(bag)
```

At first glance, this last example appears to be very similar to the situation we started with. We have however achieved two important things: a simplified structure within the view model and an `Attachable` type that guarantees correct usage of the view model from the outside.

The presented approach is unobtrusive, it consists of just one protocol and an enum. It encourages flexibility by using composition, instead of requiring a superclass. It can also be extended to support more complex use cases. For example, in the context of a collection view, the view model could support repeated attaching and detaching as cells are being reused. Another addition would be support for writing protocol extensions on `Attachable` in order to use the view model's logic before it's been attached to the view.

This is by no means the final say on Reactive MVVM, but it's an encouragement to continue exploring solutions within the paradigm.
