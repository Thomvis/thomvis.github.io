---
layout: post
title: "Compile-time dependency injection using code generation"
---

Dependency injection is a design pattern where types explicitly declare their dependencies and allow them to be set from the outside. This can either be on creation, as parameters of the initializer, or later, through public setters.

You can apply dependency injection in code yourself, or use a framework to reduce some boilerplate. In this post, we'll have a look at an approach to dependency injection that improves upon purely manual procedure, without requiring a runtime dependency. Inspired by [Dagger](https://google.github.io/dagger/), we will look at how code generation (using [Sourcery](https://github.com/krzysztofzablocki/Sourcery)) can help us injecting our dependencies.

In the examples, a `ListViewModel` will be used that depends on `APIService`, which is a singleton. Whenever an item is selected, the view model will create `DetailViewModel` instances. The latter has `PriceFormatter` as a dependency. In order for `ListViewModel` to create `DetailViewModel` instances, it has to have `PriceFormatter` as a dependency as well.

A naïve implementation using manual dependency injection looks like this:

```swift
class ListViewModel {
    
    let apiService: APIService
    let formatter: PriceFormatter    

    init(apiService: APIService, formatter: PriceFormatter) {
        self.apiService = apiService
        self.formatter = formatter
    }

    func refresh() {
        apiService.get(resource) { items in
            // do something
        }
    }

    func didSelect(item: Item) {
        let vm = DetailViewModel(formatter: formatter)
        vm.item = item
        // do something
    }
}

class DetailViewModel {
    let formatter: PriceFormatter

    var item: Item?

    init(formatter: PriceFormatter) {
        self.formatter = formatter
    }

    var price: String {
        return formatter.format(item?.price)
    }
}
```

The creation of a view model instance would look something like this:

```swift
let vm = ListViewModel(apiService: APIService.shared, formatter: PriceFormatter())
```

This implementation is already pretty nice; it's testable, flexible and decoupled. However, the way `ListViewModel` creates new instances of `DetailViewModel` doesn't sit right.

`ListViewModel` now has to keep a formatter around that it doesn't really use. If `DetailViewModel` would have had five dependencies, this would be even more problematic. And if its dependencies would change later on, you would need to make changes to `ListViewModel` (and perhaps places where `ListViewModel` is instantiated) as well. Ideally, it should have a way to create new `DetailViewModel` instances without needing to know about its dependencies. 

This problem can be solved in several ways. You could use [a](https://github.com/Swinject/Swinject) [third-party](https://github.com/square/Cleanse) [framework](https://github.com/AliSoftware/Dip) to simplify the proces. There are also aproaches that just leverage [nice](http://merowing.info/2017/04/using-protocol-compositon-for-dependency-injection/) [features](http://artsy.github.io/blog/2016/06/27/dependency-injection-in-swift/) of Swift to achieve the same. We'll look at a third solution using code generation.

Code generation gives you the benefits of a third-party framework, but can be more efficient for and focussed on your use case. The generated code will fit your project like a glove. Or like a custom tailored suit. Enough with the metaphors, let's look at how to do this.

We're using Sourcery to generate code for our dependency injection container. The container knows how to instantiate all our objects. We'll use [constructor-based injection](https://en.wikipedia.org/wiki/Dependency_injection#Constructor_injection) and annotate the initializer that should be used with `inject`. Types that should only be instantiated once, are annotated with `singleton`. Sourcery annotations are defined in comments:

```swift
// sourcery: singleton
class APIService {
    // sourcery: inject
    init() { }
}
```

Sourcery parses your code, creating a structured representation of all types, methods, parameters, annotations and more. This representation is then fed into user-defined templates that define what code should be emitted for the given representation.

We're using [Stencil](http://stencil.fuller.li/en/latest/) to write a template that defines our dependency container:

```
{% raw %}
class Container {

    {% for type in types.all %}
    public func {{type.name|lowerFirstWord}}() -> {{type.name}} {
        {% for ini in type.initializers|annotated:"inject" %}
        return {{type.name}}(
            {% for param in ini.parameters %}
            {{param.argumentLabel}}: {{param.typeName.name|lowerFirstWord}}(){% if not forloop.last%},{% endif %}
            {% endfor %}
        )
        {% endfor %}
    }

    {% endfor %}
}
{% endraw %}
```

In this template, we iterate over each type and emit a function with the name of that type (lowercasing the first word). We then loop over the type's initializers that are annotated with `inject` (there should be only one), and emit code that calls that initializer. The values that are passed as parameters to that initializer, i.e. the dependencies, are returned from methods on the container that have the name of the parameter's type, like the one we're emitting code for right now. The full template, including support for singletons and providers (explained below), can be found [here](https://gist.github.com/Thomvis/3d015cfc15e9a538ac6712ce2655ada5).

The generated code looks like this:

```swift
class Container {

    private let apiServiceQueue = DispatchQueue(label: "apiService")
    private var __apiService: APIService?

    public func apiService() -> APIService {
        return apiServiceQueue.sync {
            let res = __apiService ?? APIService()
            self.__apiService = res
            return res
        }
    }

    public func listViewModel() -> ListViewModel {
        return ListViewModel(
            apiService: apiService(),
            formatter: priceFormatter()
        )
    }

    public func priceFormatter() -> PriceFormatter {
        return PriceFormatter()
    }

}
```

The code is written to a file whenever the template or source changes. That file should be added to your project, but is not meant to be edited by hand. Once that's done, you can instantiate the container and ask it for an instance of `ListViewModel`:

```swift
let container = Container()
let listViewModel = container.listViewModel()
```

This is a good starting point, but it's not practical to use the container whenever you need to create an instance of a type that it manages. You would have to make it a singleton or pass it as a dependency itself. Instead, a type can declare `Provider<E>` as a dependency. It's a simple closure that returns an instance of type `E`.

```swift
public typealias Provider<E> = () -> E
```

If injected by the container, a provider returns instances managed by the container. Consequently, the provider of a singleton type would always return the same instance, while other providers would always return a new one.

We can update our view model to take a `Provider<DetailViewModel>` as a dependency:

```swift
class ListViewModel {
    
    // ...
    let detailProvider: Provider<DetailViewModel>

    init(apiService: APIService, @escaping Provider<DetailViewModel> detailProvider) {
        // ...
        self.detailProvider = detailProvider
    }

    // ...

    func didSelect(item: Item) {
        let vm = detailProvider()
        // ...
    }
}
```

The generated code is instantaneously updated accordingly:

```swift
public func listViewModel() -> ListViewModel {
    return ListViewModel(
        apiService: apiService(),
        detailProvider: detailViewModel
    )
}
```

Every function of the container matches the signature of a provider for the return type, so it can be easily passed whenever a provider is requested (`detailViewModel` in this case). Since the generated methods in the container take into account whether a type was defined as `singleton`, the providers will as well.

The `ListViewModel` no longer needs to know about the `DetailViewModel`'s dependencies. If the dependencies of the latter would change, `ListViewModel` would not have to be updated. That's quite elegant!

The result of this experiment in compile-time dependency injection has its limitations, but it does illustrate that code-generation is an compelling approach to solving existing problems or improving existing patterns. It has gained widespread use in other communities, so it's interesting to see if the same could happen to Swift.
