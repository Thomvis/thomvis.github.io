---
layout: post
title: "A Precursor to Reactive Programming"
---

I started doing reactive programming 1.5 years ago. I could point to the exact day: a significant and conscious paradigm shift. At the same time, the switch to reactive programming was part of the constant drive to find better ways to write good code.

I feel like I can write better code using reactive programming. What 'good code' is, is of course subjective. We might not share the same principles, values or rules. And even if we do, we can debate about how well a certain piece of code implements those principles.

It's useful to differentiate between principles and the means by which those principles are implemented. We can vary one of them, while keeping the other constant, and compare the resulting code. For this post, I want to pick a single principle and look at two possible implementations: imperative and reactive.

The principle that I want to focus on is that of side-effect isolation. Side-effects are changes to the state outside of the scope that makes the change. For example, a certain function could perform an API request, change state of a singleton, update a CoreData entity or change the UI as a side-effect.

It is tempting to sprinkle side-effects all over your code, but it will make your code harder to understand and test. By isolating side-effects, you increase the predictability and flexibility of your code. I've tried, and achieved, that in imperative code and reactive programming.

To illustrate the issue, I will use simplified code from [a shopping app](https://www.highstreetapp.com). Imagine a detail screen where the user can choose between colors of a product and stock information about the availability of all colors is refreshed periodically. The state of the 'buy button' depends on the availability of the currently selected color.

```swift
func stockChanged(_ stock: Stock) {
    self.stock = stock
    evaluateBuyButtonState()
}

func colorChanged(to color: UIColor) {
    self.color = color
    evaluateBuyButtonState()
}

func evaluateBuyButtonState() {
    buyButton.enabled = stock.canBuy(color)
}
```

This code achieves side-effect isolation to a certain extent. The code reads as two functions that process input and one that performs a side-effect. In practise, calling either `stockChanged` or `colorChanged` will trigger side-effects, because they directly call `evaluateBuyButtonState`.

To achieve true isolation of side-effects and make the code better testable, the direct invocation of the evaluation function would need to be replaced by a delegate call or a callback closure. The added indirection would make it possible to replace the side-effect inducing code with a mock implementation during tests.

The changes needed to achieve better testability would add significant complexity to the imperative implementation. In the past, I have therefore often settled on a middle ground similar to what is depicted in the code above.

Let's have a look at how the same scenario could be implemented using reactive programming:

```swift
let canBuy: Observable<Bool> = Observable.combineLatest(stock, color) {
    $0.canBuy($1)
}

canBuy.subscribe(onNext: { canBuy in
    buyButton.enabled = canBuy
})
```

`stock` and `color` are two Observables that emit the latest stock information and color selection. I combine those two into an observable that emits a boolean whenever either stock or color changes. 

The side-effect is contained to the closure passed to the `subscribe` method. You'll be able to put almost all side-effects in `subscribe` closures. (Other places that make sense to have sid-effects include the subscription handler when creating an observable and the `do` operator.) By making this a rule, or convention, the code becomes easy to understand. Combined with the fact that observables are easily composable, the code becomes easy to test.

The rules around where to put side-effects (e.g. `subscribe`) and where not to (e.g. `map`) are currently not enforcable by the language. You can use the [`@effects` annotation](https://github.com/apple/swift/blob/9ce3df106ebfc18e49d41730c31c7211bbab62dd/docs/HighLevelSILOptimizations.rst#effects-attribute) to tell the optimizer about side-effects or dependencies, allowing it to emit more efficient code, but it is not enforced by the compiler.

If you try to achieve isolation of side-effects in your code, consider giving reactive programming a try to see how it can help implementing this principle.

[This talk](https://www.youtube.com/watch?v=A0VaIKK2ijM) by Brandon Williams prompted me to write about this.


