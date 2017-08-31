---
layout: post
title:  "Property and Method Injection on Storyboard Instantiated View Controllers"
categories: iOS Swift
disqus: true
---

In the debate between using programmatic views and storyboards, perhaps the two of the biggest drawbacks to using storyboards is the lack of static typing and the inability to use constructor injection.


In Andyy Hope's blog post [Swift UIStoryboard Protocol:
Because String literals are so yucky](https://medium.com/swift-programming/uistoryboard-safer-with-enums-protocol-extensions-and-generics-7aad3883b44d), he gives a nice little way to conveniently instantiate view controllers from storyboards without relying on String literals at the point of instantion. You end up with these methods:

{% highlight swift %}
import UIKit

extension UIStoryboard { 

    // MARK: - Convenience Initializers
    
    convenience init(storyboard: Storyboard, bundle: Bundle? = nil) {
        self.init(name: storyboard.filename, bundle: bundle)
    }
    
    // MARK: - Class Functions
    
    class func storyboard(_ storyboard: Storyboard, bundle: Bundle? = nil) -> UIStoryboard {
        return UIStoryboard(name: storyboard.filename, bundle: bundle)
    }
    
    // MARK: - View Controller Instantiation from Generics
    
    // `where T: StoryboardIdentifiable` is also required but is implied in UIViewController+StoryboardIdentifiable
    func instantiateViewController<T: UIViewController>() -> T  {
        guard let viewController = self.instantiateViewController(withIdentifier: T.storyboardIdentifier) as? T else {
            fatalError("Couldn't instantiate view controller with identifier \(T.storyboardIdentifier) ")
        }
        return viewController
    }
}

protocol StoryboardIdentifiable {
    static var storyboardIdentifier: String { get }
}

extension StoryboardIdentifiable where Self: UIViewController {
    static var storyboardIdentifier: String {
        return String(describing: self)
    }
}

// MARK: UIViewController+StoryboardIdentifiable

extension UIViewController : StoryboardIdentifiable { }
{% endhighlight %}

These protocols give a nice way to get some pseudo-static typing along with a sensible naming scheme for storyboards and a useful leverage of generics. There are other tools such as [R.swift](https://github.com/mac-cain13/R.swift/) which can also generate statically-typed declarations for things such as storyboards, fonts and assets.

However, we still have to property inject or method inject any dependencies on the instantiated view controller. Since we don't have access to the view controller's initializer when utilizing storyboards, we're stuck with those two methods and constructor injection isn't an option, but we can do the next best thing and attach a closure to our instantiation method.

Let's have our `instantiateViewController` function take in a closure parameter

{% highlight swift %}
func instantiateViewController<T: UIViewController>(with injectionMethod: ((T) -> Void)? = nil) -> T  {
    guard let viewController = self.instantiateViewController(withIdentifier: T.storyboardIdentifier) as? T else {
        fatalError("Couldn't instantiate view controller with identifier \(T.storyboardIdentifier) ")
    }
    injectionMethod?(viewController)
    return viewController
}
{% endhighlight %}

Now when we instatiate our view controllers, we can resolve any dependencies in the closures like so:

{% highlight swift %}
let vc: ExampleViewController = UIStoryboard.storyboard(.example).instantiateViewController { (vc) in
    vc.configure(foo: bar)
}
return vc
{% endhighlight %}

Here, `configure` can be a function where the view controller resolves any dependencies and can be "constructed" with any needed parameters. Property injection can also be done in the closure. A benefit to this is that the entire configured view controller can be passed as a first class object and handled as a single unit. 

There's still one big downside to this approach: injected properties on the UIViewController class still must have an optional type. If properties are added and the view controller is configured from multiple places within the application, unless the method signature on a class using method injection is updated, you run the risk of forgetting to inject the property (a usual downside to non-constructor dependency injection). 

We could also extend this to allow for view controllers that configure their own dependencies. Taking a real basic protocol for injection:

{% highlight swift %}
protocol Injectable: class {
    func injectProperties()
}
{% endhighlight %}

We can use that to call the `injectProperties()` function on any class that conforms to the Injectable protocol. It allows us to modify our `instantiateViewController` function to let the view controll resolve its own dependencies.

{% highlight swift %}
func instantiateViewController<T: UIViewController>(with injectionMethod: ((T) -> Void)? = nil) -> T  {
    guard let viewController = self.instantiateViewController(withIdentifier: T.storyboardIdentifier) as? T else {
        fatalError("Couldn't instantiate view controller with identifier \(T.storyboardIdentifier) ")
    }
    
    if let injectableViewController = viewController as? Injectable {
        injectableViewController.injectProperties()
    }
        
    injectionMethod?(viewController)
    return viewController
}
{% endhighlight %}

In the `injectProperties()` function, the view controller could query a provider or a service locator class where dependencies are registered and set its instance variables accordingly.

In the absense of Apple [providing a way to initialize a view controller with an NSCoder instance and the view controllers dependencies](http://holko.pl/2016/12/14/storyboards-dependency-injection/), using attached closers with method injection, and instantiating view controllers from common locations such as flow controllers, might be the best we'll get for projects using UIStoryboards.
