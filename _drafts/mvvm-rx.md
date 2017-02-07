---
layout: post
title: "Reactive MVVM"
---

There are many ways to implement MVVM or to put RxSwift to use, let alone combine the two. It comes down to 


```swift
// 1. initialize the view model with dependencies & model
let vm = ViewModel(apiClient: apiClient, database: database, model: model)

// 2. provide the view model with inputs from the view
vm.setInputs(button.rx.tap, usernameField.rx.text)

// 3. subscribe the view to the outputs from the view model
vm.buttonEnabled().subscribe(onNext: {
    button.setEnabled($0)
}).addDisposableTo(bag)
```

You probably won't (want to) find these lines of code so close together in a serious code base. It presumes a single context where the app's infrastructure, models and view are within reach. Such a context is likely to be tightly coupled to other parts of the app and will prove to be hard to test as well.

A second issue with this approach is the implicit requirement on the order in which step 2 and 3 are executed. This becomes apparent as we look at a typical implementation of the view model:

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

Another solution is to make `usernameText` a [Subject](http://reactivex.io/documentation/subject.html). The Subject can be created at initialization and after that, the order in which `setInputs` and `buttonEnabled` are called doesn't really matter. The downside is however that you'll know have to manage a subscription on `username` in `setInputs` and deal with additional state that comes with a Subject. The latter being the reason why you might have heard that [Subjects should be avoided](http://introtorx.com/Content/v1.0.10621.0/18_UsageGuidelines.html). They definately have their use, but I think we can do better here.

As we've been fiddling with Optionals and Subjects, we've been avoiding the real issue here: we're dealing with an object that has a transient and inherently uncomfortable state. The use of multiple Optionals at a single level are a symptom of this. A solution can be to look one level up.

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

The power of the solution that we're looking for will come, in part, from the fact that it doesn't care (i.e. make no assuptions) on the inner workings of the view model. The solution can be generic. This does require us to quickly formalize what a view model looks like from the outside and how it can be initialized:

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

Revisiting the initial example, we can now create a view model like so:

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

We've reduced the state space significantly: the original approach with two Optionals had four possible states, while our final solution only has two. With more complex (realistic) view models, the original approach could easily yield dozens of states, while the final solution remains at a stable two states.

<!-- , not unlike the process of untangling knitting yarn -->