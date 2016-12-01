---
layout: post
title: "MVVM Rx"
---

- introduction: rx and mvvm are a good fit
- separation:
    - model: creates observables
    - view model: combines & transforms observables
    - view: subscribes to and creates observables

This approach to reactive view models was developed at [Highstreet mobile retail](http://www.highstreetapp.com).

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



You'll end up with lots of optionals: an optional `viewModel` in the view controller and optional properties (or Subjects) in the view model where the bindings with the view go. With every optional comes the decision of what to do if it is nil. Subjects should not be used when Observables suffice.

Here are several candidates for where to create the view model:

- the View Controller's `init`: the view does not yet exist at this point, so this won't work.
- the View Controller's `viewDidLoad`: the view is here, nice and fresh, but the dependencies and model are in 'way too deep'. I don't want the view (controller) to know about the dependencies and the model.
- the creator of the View Controller: this might be a parent view model. The view definately does not yet exist at this point.

Neither of these candidates seem suitable. The ingredients seem to become available in two steps: first the dependencies and the model, then the bindings. We don't want to get rid of the ViewModel initializer that takes all three ingredients, because we'll get optionals and Subjects in return. Instead, let's wrap our view model in a type that represents the two states it can be in: either detached (with just the dependencies and the model) or attached (all three, i.e. the full view model).

```swift
// LoginViewModel.swift

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

A view model is created in detached state, by passing...

Example:

```swift
var viewModel: Attachable<ExampleViewModel> = .detached(deps, model)
if let vm = viewModel.bind(bindings) {
    // vm is an ExampleViewModel
}
```

![A login screen in the Scotch & Soda app](/media/login.png){:width="50%"}

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

The login dependency is a function that takes two strings (the username and password) and returns an Observable, representing the asynchronous login process. Instead of passing in a function, we could also have passed a `UserSessionController`. By passing in a function, you prevent tight coupling and make the view model easier to test: it is easier to provide a test function than to mock a whole `UserSessionController`.

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
// LoginViewModel.swift

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
// LoginViewModel.swift

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

Advanced:

- rebinding vm's
- navigation between view models
- writing part of the VM's logic in an extension of Attachable to be able to use the VM before it is attached.