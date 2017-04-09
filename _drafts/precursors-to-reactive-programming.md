---
layout: post
title: "A Precursor to Reactive Programming"
---

I started doing reactive programming 1.5 years ago. I could point to the exact day: a significant and conscious paradigm shift. At the same time, the switch to reactive programming was part of the constant drive to find better ways to write good code.

`good code = principles x implementation`

`implementation = effort x tools x tool proficiency`

`tools = Swift x RxSwift x MVVM x Xcode x ...`

Was my first reactive code really better than the code I wrote before? To be honest, it hasn't aged incredibly well. Not as well as the imperative code I wrote before that. So why did I switch?e

These principles change at a slower pace; it is a more fundamental proces fueled by knowledge, experience and inspiration. Simultaneously, we find new ways to implement those principles. We'll switch tools, programming paradigms and even languages, trying to implement the same principles, but in a better way. It's also fun to do and it inspires us to improve our principles.

I want to give an example of one of those principles and how the implementation changed as I switched to reactive programming. Let's first look at a typical imperative implementation:

As I got better at reactive programming, my code would to a greater degree reflect the principles of good code.

That's how I feel I can currently write my best code. A big part of that is how reactive programming helps me separate data flow from side-effects. While easily mixed together, data flow and side-affects are best kept apart in code that is easy to understand and test.

Looking back at the imperative code I wrote in the months before I started using reactive programming, I can now identify attempts to achieve that desired separation.

Let's look at an example of that:

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



Note that this is already a step up from a naïve implementation where no effort is made to separate data flow from side-effects. In that implementation, there would be no `evaluateBuyButtonState()` and the buy button would be updated in both `stockChanged` and `colorChanged`.



```swift
Observable.combineLatest(stock, color) {
    $0.canBuy($1)
}.subscribe(onNext: { canBuy in
    buyButton.enabled = canBuy
})
```

If you are striving for separation between data flow and side-effects, consider giving reactive programming a try to see how it can help implementing your principles.

[This talk](https://www.youtube.com/watch?v=A0VaIKK2ijM) by Brandon Williams.


