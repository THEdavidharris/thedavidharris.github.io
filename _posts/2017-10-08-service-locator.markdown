---
layout: post
title:  "Let's Build a Service Locator using Swift Generics"
categories: iOS Swift
disqus: true
---

At the core of most dependency injection frameworks used throughout the industry is the concept of the dependency container. The dependency container can register abstract definitions to concrete implementations, allowing any class to query the container to resolve a dependency type. Having a dependency container also allows for dependecy objects to have some "scope" for the object lifetime, which lends to all sorts of possible patterns and anti-patterns. This container is inserted to and queried via register and resolve functions on some sort of service locator class.

Before going any further: the service locator pattern is not without significant downsides. Most significantly, it lacks compile-time checking and requires runtime configuration. Because of this, standard initializer injection should be preferred, but when not possible the service locator can be of service, for lack of a better word.

Swift's generics make constructing a dependency container pretty simple. At the core of the service locator is our dictionary of services:

{% highlight swift %}
var container: [ObjectIdentifier:Any]
{% endhighlight %}

Swift's `ObjectIdentifier` [uniquely identifies any class instance or metatype](https://github.com/apple/swift/blob/master/stdlib/public/core/ObjectIdentifier.swift#L32). It's a suitable key for our dictionary since we only want to have one registry per object or type. The values in our dictionary are of `Any` type. The general usage of the container is to register an implementation for an interface, so when we query the container with an `ObjectIdentifier` for a protocol metatype, we're returned the registered implementation.

We can do this by registering recipes for interfaces, as well as class objects, with our register function:

{% highlight swift %}
func register<T>(_ recipe: () -> T) {
  container[ObjectIdentifier(T.self)] = recipe
}
{% endhighlight %}

The register function takes is templated to deal with any metatype, and takes in a recipe to create a class. It gives a way to register services as protocols, as well as just register concrete types if that's desired as well.

{% highlight swift %}
ServiceLocator.register({ SomeServiceImp() })
ServiceLocator.register({ SomeServiceImp() as SomeService })
{% endhighlight %}

The first register will register a function to the class type, the second to a protocol type (assuming conformance).

Now that we've registered something to the container, we want a generic resolve function we can call to resolve any of the metatypes or instances we've registered.

{% highlight swift %}
private func resolve<T>() -> T {
    if let recipe = container[ObjectIdentifier(T.self)] as? () -> T {
        return recipe()
    } else {
        fatalError("Could not find registered definition for \(T.self)")
    }
}
{% endhighlight %}

We'll use generics again to pick up the type we want to query for, so using this function would look like:

{% highlight swift %}
let service: SomeService = ServiceLocator.resolve()
{% endhighlight %}

And if we've registered a definition for the `SomeService` like we did above, the service variable will now be an instance of `SomeServiceImp`.

This is where the runtime issues of the service locator pop up: if there's no registered recipe for the type that the service locator is being asked to resolve, we throw an error. Anything asked be to resolved must always be registered first.

Bundling these functions into a `ServiceLocator` class with a static container gives us a real simple way to register dependencies in some application root, and resolve dependencies on classes when needed. For tests, we can register mock implementations to the same metatypes our unit-tested classes are trying to resolve. It's also possible to extend this pattern to register concrete instances instead of recipes, and give these concrete instances a lifetime scope.

Despite the downsides, the service locator pattern still has its uses, even as a fallback. In instances where protocol definitions are implementations are across targets, it can prevent importing entire frameworks by registering the recipe in the framework that has the implementation, and querying for the metatype in the framework that uses the abstraction. It can also be used to resolve dependencies in classes where there is no access to the initializer, such as UIViewControllers instantiated via storyboards.