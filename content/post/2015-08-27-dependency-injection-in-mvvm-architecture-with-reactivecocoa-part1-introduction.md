+++
date = "2015-08-27T13:45:30+09:00"
slug = "dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-1-introduction"
tags = ["swinject", "dependency-injection", "mvvm", "reactivecocoa", "swift"]
title = "Dependency Injection in MVVM Architecture with ReactiveCocoa part 1: Introduction"

+++

In [the last blog post](/post/dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-2/), we developed an app demonstrating dependency injection with [Swinject](https://github.com/Swinject/Swinject) in a simple architecture. In the series of blog posts from today, we are going to develop an app taking advantage of dependency injection in [MVVM (Model-View-ViewModel)](https://en.wikipedia.org/wiki/Model_View_ViewModel) architecture with [ReactiveCococa](https://github.com/ReactiveCocoa/ReactiveCocoa).

Through the series of the blog posts, you will learn:

1. How to setup an Xcode project to explicitly represent MVVM architecture.
2. How to inject dependencies in the architecture with Swinject.
3. How to use ReactiveCocoa to propagate events from Model to ViewModel and ViewModel to View.

The blog posts concentrate on the practical development using dependency injection, Swinject, MVVM and ReactiveCocoa. Please refer to other articles to learn the details of them. Recommended articles will be referenced later in the blog posts.

The example app we are going to develop asynchronously searches, downloads and displays images obtained from [Pixabay](https://pixabay.com) via [its API](https://pixabay.com/api/docs/), as shown in the GIF animation below.

![SwinjectMVVMExample ScreenRecord](/images/post/2015-08/SwinjectMVVMExampleScreenRecord.gif)

The source code used in the blog posts is available at [a repository on GitHub](https://github.com/Swinject/SwinjectMVVMExample).

## MVVM

MVVM (Model-View-ViewModel) is an architecture or pattern to make dependencies of components simple as View depends on ViewModel and ViewModel depends on Model linearly. Its event flow is linearly, in reverse, from Model to ViewModel and ViewModel to View.

![MVVM Diagram](/images/post/2015-08/Diagram-MVVM.png)

On the other hand, in MVC (Model-View-Controler) architecture or pattern, Controller depends on Model and View, and its event flow is from Model to Controller and View to Controller.

![MVC Diagram](/images/post/2015-08/Diagram-MVC.png)

The problem of MVC is that Controller tends to get large and complex as a project evolves because it has to care of both Model and View, which mean everything. Actually MVC is a good pattern in web applications with the support of frameworks such as [Rails](http://rubyonrails.org) or [ASP.NET MVC](http://www.asp.net/mvc), but in iOS apps MVC often makes monolithic and hard-to-maintain code.

For the disadvantage of MVC, MVVM is getting popular to develop mobile apps or desktop apps. In iOS apps, the "View" of MVVM is composed of "View" (UIView) and "ViewController" (UIViewController). View logic, e.g. a value `1000` should be displayed as `"1,000"`, is implemented in ViewModel. View simply uses values provided by ViewModel to display. Model is responsible for business logic. Because of the separation of responsibilities, an iOS app in MVVM architecture is easier to test.

![iOS MVVM Diagram](/images/post/2015-08/Diagram-MVVM-iOS.png)

### References

- [Introduction to MVVM](https://www.objc.io/issues/13-architecture/mvvm/)
- [ReactiveCocoa and MVVM, an Introduction](http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/)
- [MVVM Tutorial with ReactiveCocoa: Part 1/2](http://www.raywenderlich.com/74106/mvvm-tutorial-with-reactivecocoa-part-1)

## ReactiveCocoa

MVVM propagates events from Model to ViewModel and ViewModel to View. We are going to use ReactiveCocoa to handle the events. The framework provides APIs for composing and transforming event streams. Without ReactiveCocoa, events are represented by delegate methods, callback closures, `UIControl` actions, or KVO (Key-Value Observation). We had to write a different way of handling to each type of events. With ReactiveCocoa, an event is represented by `Event` and event streams are represented by `Signal` or `SignalProducer`. We can handle the events in the same abstracted way regardless of the original source of events. We will take advantage of the simplicity of the events with ReactiveCocoa. If you are new to ReactiveCocoa, it is recommended to read the following articles before proceeding to the next section of this blog post.

### References

- [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
- [Framework Overview](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/FrameworkOverview.md)
- [A First Look at ReactiveCocoa 3.0](http://blog.scottlogic.com/2015/04/24/first-look-reactive-cocoa-3.html)

## Requirements

- Xcode 7 beta 6
- [Carthage](https://github.com/Carthage/Carthage) 0.7.5 or later
- [Pixabay](https://pixabay.com/api/docs/) API username and key

We are going to develop the example program in Swift 2 with Xcode 7 although they are still beta versions. To use with Swift 2, we are going to use development version of ReactiveCocoa in [swift2 branch of its repository](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/swift2). Notice that ReactiveCocoa 3.0 has functional style APIs like `|> map` or `|> flatMap`, but APIs in swift2 branch are in protocol oriented style like `.map()` or `.flatMap()`. Since swift2 branch is still in development, the APIs might be changed in the future.

You can get a free API username and key at [Pixabay](https://pixabay.com/). First, sign up and log in there. Then, access its [API documentation page](https://pixabay.com/api/docs/). Your API username and key will be displayed in "Request parameters" section.

## Conclusion

MVVM has the simple and linear dependencies of View on ViewModel and ViewModel on Model, and event flows from Model to ViewModel and ViewModel to View. ReactiveCocoa turns various kind of events, such as delegate methods or callback closures, into a single type `Event` of events. For the simplicity, an iOS app in MVVM architecture with ReactiveCocoa is easier to maintain and test. From the next blog post, step-by-step development with MVVM, ReactiveCocoa, dependency injection and Swinject will be demonstrated.
