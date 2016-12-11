---
layout: post
title: "Reactive MVVM"
---

RxSwift is a great tool for building your MVVM architecture. The three components of MVVM map nicely to concepts from RxSwift. The model layer is where Observables are created, the view model combines and transforms Observables and the view Observes the result. There's no one right way to use RxSwift or implement MVVM. However, the details of your implementation and the boundaries you draw determine how concise and testable your code will be.

In this post, I'll discuss an approach to _Reactive MVVM_ that I developed at [Highstreet mobile retail](http://www.highstreetapp.com). It has served us well over the past year and I hope you'll find it equally helpful. The approach involves a tiny bit of shared 'library' code, describing a common interface for view models, which should provide guidence to setting the right boundaries and creating a clear separation of concerns.

Creating and using a view model is usually a three step process: 

1. initialize the view model 
2. bind the view model to the view inputs
3. bind the view to the view model outputs

Depending on your approach, these steps could take place at different points in time or code. To accomodate the intermediate states where part of the view or view model are not yet set-up completely, you will end up using optionals or Subjects. The latter is a concept in Reactive programming that is tempting to use as an optional Observable. I wanted to find an approach that didn't require unwrapping optionals or dealing with Subjects all the time.

```swift
// LoginViewModel.swift

public protocol ViewModel {
    associatedtype Dependencies
    associatedtype Model
    associatedtype Bindings

    init(dependencies: Dependencies, model: Model, bindings: Bindings)
}
```

Dependencies are other parts of the app this view model needs to know about. Examples are the networking layer, the parent view model and analytics trackers.

The Model is the model in the traditional sense and/or a representation of the (initial) view state.

Bindings are the parts of the view that the view model needs to know about. Examples are button taps, scroll events and form field contents.

Here are several candidates for where to create the view model:

- the View Controller's `init`: the view does not yet exist at this point, so this won't work.
- the View Controller's `viewDidLoad`: the view is here, nice and fresh, but the dependencies and model are in 'way too deep'. I don't want the view (controller) to know about the dependencies and the model.
- the creator of the View Controller: this might be a parent view model. The view definately does not yet exist at this point.

None of these candidates seem suitable. The ingredients seem to become available in two steps: first the dependencies and the model, then the bindings. We don't want to get rid of the ViewModel initializer that takes all three ingredients, because we'll get optionals and Subjects in return. Instead, let's wrap our view model in a type that represents the two states it can be in: either detached (with just the dependencies and the model) or attached (all three, i.e. the full view model).

```swift
// Attachable.swift

public enum Attachable<VM: ViewModel> {
    
    case detached(VM.Dependencies, VM.Model)
    case attached(VM)
    
    public mutating func bind(_ bindings: VM.Bindings) -> VM? {
        if case let .detached(dependencies, model) = self {
            let vm = VM(
                dependencies: dependencies, 
                model: model, 
                bindings: bindings
            )
            self = .attached(vm)
            return vm
        }
        return nil
    }
    
    public var viewModel: VM? {
        if case let .attached(vm) = self {
            return vm
        }
        return nil
    }
}
```

A view model is created in detatched state, requiring only the dependencies and model. The detached view model is passed to the view (controller) and when the view is ready to be bound, the bindings are passed to the `bind` method. This method is mutating, because the Attachable changes from detached to attached. At the call site, this looks like this:

Example:

```swift
var viewModel: Attachable<ExampleViewModel> = .detached(deps, model)
if let vm = viewModel.bind(bindings) {
    // vm is an ExampleViewModel
}
```

`bind` returns the newly created view model instance as a convenience. It immediately be unwrapped and used. Typically, inside the `if let` body, subscriptions to the view model's outputs are set-up. 

For the remainder of this post, we'll look at the interesting code that goes into creating a login form view. Simple enough, and also familiar if you've been reading other posts or books on Reactive programming. Here's a screenshot:

![A login screen in the Scotch & Soda app](/media/login.png){:width="50%"}

Let's begin by creating the view model. 

```swift
// LoginViewModel.swift

public class LoginViewModel: ViewModel {
    
    let bindings: Bindings

    init(dependencies: Dependencies, model: (), bindings: Bindings) {
        self.bindings = bindings
    }

    struct Dependencies {
        let login: (String, String) -> Observable<Bool>
    }

    struct Bindings {
        let usernameText: Observable<String>
        let passwordText: Observable<String>
        let buttonTaps: Observable<Void>
    }

}
```

The view model has one dependency. It is a function that takes two strings (the username and password) and returns an Observable, representing the asynchronous login process. Instead of passing in a function, we could also have passed a `UserSessionController`. By passing in a function, you prevent tight coupling and make the view model easier to test: it is easier to provide a test function than to mock a whole `UserSessionController`.

For this example, we won't use a model, therefore we define it as Void. In a real-world case, the model could contain the initial contents of the form, e.g. to prefill a known username.

The bindings consist of Observables representing the text in the two input fields and the clicks on the login button.

```swift
// LoginViewController.swift

public class LoginViewController: UIViewController {
    
    var viewModel: Attachable<LoginViewModel>

    public init(viewModel: Attachable<LoginViewModel>) {
        self.viewModel = viewModel
    }

    public override func viewDidLoad() {
        super.viewDidLoad()

        let bindings = LoginViewModel.Bindings(
            usernameText: Reactive(view.usernameTextField).text,
            passwordText: Reactive(view.passwordTextField).text,
            buttonTaps: Reactive(view.button).taps
        )

        if let vm = viewModel.bind(bindings) {
            view.bind(to: vm)
        }
    }
}
```

```swift
// LoginViewModel.swift, inside LoginViewModel

let bag = DisposeBag()
let form: Observable<(String, String)>

init(dependencies: Dependencies, model: (), bindings: Bindings) {
    self.bindings = bindings

    self.form = Observable.combineLatest(
        bindings.usernameText, 
        bindings.passwordText
    ) { ($0, $1) }.shareReplay(1)

    let login = bindings.buttonTaps.withLatestFrom(form) { form in
        dependencies.login(form.0, form.1)
    }.concat()

    login.subscribe().addDisposableTo(bag)
}
```

- init is used to create observables that define inner working or represent intermediate results that might be used by multiple outputs, to prevent 'expensive' side-effects from happening twice
- in this case we subscribe to the login observable, tying it to the lifecycle of the VM. in other cases it might make more sense to tie it to some external factor

```swift
// LoginViewModel.swift, inside LoginViewModel

var buttonEnabled: Observable<Bool> {
    return form.map { form in
        return !form.0.isEmpty && !form.1.isEmpty
    }
}
```

```swift
// LoginView.swift

public class LoginView: UIView {
    
    // textfield and button properties omitted
    let bag = DisposeBag()

    func bind(to viewModel: LoginViewModel) {
        viewModel.buttonEnabled.subscribe(onNext: { [weak self] e in
            self?.button.isEnabled = e
        }).addDisposableTo(bag)
    }

}
```

This approach to view models also enables some more advanced patterns. For example, in the case of a collection view you might want to detach and attach a single view model to different views as they are being reused. In other situations, you might want to use part of the view model logic before it is attached to a view. You could write that logic in a conditional extension on Attachable to achieve that. I might get back to these topics in a future post.

