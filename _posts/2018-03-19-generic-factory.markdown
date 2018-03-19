---
layout: post
title:  "Building a Generic Factory in Swift"
categories: iOS Swift
disqus: true
---

The factory pattern is a great tool that abstracts away the constructing of objects without required knowledge of the concrete class type to create, or access to the class initializers. By instantiating dependencies with factory methods, a program gets not only the benefits of potentially smaller constructors as depedency instantiation can be consolidated, but also gives an easy option for programming to abstractions and ease of dependency injection. It also allows for the lazy instantiation of parameters at some point other than class construction, as well as being able to reinstatiate new or multiple instances.

In Swift, factories can be created by starting with a protocol.

{% highlight swift %}
protocol VehicleFactory {
    func makePickup(with color: UIColor) -> Pickup
    func makeConvertible() -> Convertible
}
{% endhighlight %}

The concrete implementations can then be provided in a conforming class.

{% highlight swift %}
class VehicleContainer {
    func makePickup(with color: UIColor) -> Pickup {
        return Pickup(color: color)
    }

    func makeConvertible() -> Convertible {
        return Convertible()
    }
}
{% endhighlight %}

Classes could then be created with the VehicleFactory as a parameter, and be able to create Vehicles and Pickups by simply calling the `make` functions, without having to know about any other details other than the factory functions.

It's possible to make the injection of mocked dependencies easy by reimplementating the protocol to provide mocked instances.

{% highlight swift %}
class FakeVehicleContainer {
    func makePickup(with color: UIColor) -> Pickup {
        return FakePickup(color: color)
    }

    func makeConvertible() -> Convertible {
        return FakeConvertible()
    }
}
{% endhighlight %}

Not too bad. However, there are some pitfalls, namely that an entire separate class must be maintained and separate functions must be added for every dependency, and code organization must be taken into account. While alternatives might not necessarily be cleaner in all cases, they can be interesting to use in some cases.

### Injecting a closure

In order to achieve some of the same benefits, instead of an entire factory, it's possible to just inject a closure that has the same representation as the factory function.

{% highlight swift %}
class VehicleController {
    typealias PickupCreator (UIColor) -> Pickup

    let firstPickup: Pickup
    let secondPickup: Pickup

    init(pickupCreator: PickupCreator) {
        self.firstPickup = PickupCreator(.blue)
        self.secondPickup = PickupCreator(.black)
    }
}

let vehicleController = VehicleController { color in
    Pickup(color: color)
}
{% endhighlight %}

It's possible to pass any closure into the function now, as long as it takes in a color parameter and returns a Pickup object or subclass, without the need for a factory class.

Let's go one step further and genericize this, starting with an open base class.

### The Generic Factory

{% highlight swift %}
/// Utility that instantiates a class of type DependencyType
open class Builder<DependencyType> {

    /// Closure to instantiate the dependency
    public typealias DependencyCreator = () -> DependencyType

    /// The closure to create the dependency used for this builder to build the unit.
    public var dependencyCreator: DependencyCreator

    /// Initializer.
    ///
    /// - parameter dependencyCreator: The closure to create the dependency used for this builder, typically a view model.
    public init(dependencyCreator: @escaping DependencyCreator) {
        self.dependencyCreator = dependencyCreator
    }

    /// Creates an instance of the dependency type
    ///
    /// - Returns: an object of DependencyType
    public func create() -> DependencyType {
        return dependencyCreator()
    }
}
{% endhighlight %}

By creating instances of this class, we can create a mini-factory of sorts for the `DependecyType`

{% highlight swift %}
let vehicleBuilder = Builder<Vehicle> { Vehicle() }
let firstVehicle = vehicleBuilder.create()
let secondVehicle = vehicleBuilder.create()
{% endhighlight %}

Here, `firstVehicle` and `secondVehicle` will be two distinct instances. We could also subclass the original class and provide it a convenience initializer if we wanted further abstraction.

{% highlight swift %}
class VehicleBuilder: Builder<Vehicle> {
    convenience init() {
        self.init { Vehicle() }
    }
}

let vehicleBuilder = VehicleBuilder()
let firstVehicle = vehicleBuilder.create()
let secondVehicle = vehicleBuilder.create()
{% endhighlight %}

It then becomes easy to sublass the VehicleBuilder and override the `init` to return a FakeVehicle for testing purposes.

{% highlight swift %}
class FakeVehicleBuilder = VehicleBuilder { FakeVehicle() }
{% endhighlight %}

It's not necessarily cleaner in all instances than creating a separate factory, but it's a nice way to create a simple factory pattern without too much boilerplate. However, the above version only works for classes that don't require any dependencies to instantiate. Such a Builder would have to look something like this:

{% highlight swift %}
open class PayloadBuilder<Dependency, Payload> {

    private let dependencyClosure: (Payload) -> Dependency

    public init(dependencyCreator: @escaping (Payload) -> Dependency) {
        self.dependencyClosure = dependencyCreator
    }

    public func create(payload: Payload) -> Dependency {
        return dependencyClosure(payload)
    }
}

extension PayloadBuilder where Payload == Void {
    func create() -> Dependency {
        return dependencyClosure(())
    }
}
{% endhighlight %}

Here the class has a second generic, which tells the depedency creator what parameters. For a single parameter dependency, it's pretty straightforward.

{% highlight swift %}
let pickupBuilder = PayloadBuilder<Pickup, UIColor> { (color) in
    Pickup(color:color)
}

let bluePickup = pickupBuilder.create(payload: .blue)
{% endhighlight %}

For passing multiple depdencies, the `Payload` type can be formed as a tuple of objects such as `(color: UIColor, licensePlate: String)` or by passing a struct containing the initialization parameters. It definitely gets messier the larger the initializer, a pitfall of the standard factory pattern (and a poor practice in general) as well as this mini-generic factory.

This base builder class has been a great addition in the codebase to make it easier to create testable code via dependency injection through these mini-factories, without having to create factory classes that group unrelated objects. It's also an interesting exploration of Swift closure and generics, and takes advatage of the strong type system to ensure type safety throughout.
