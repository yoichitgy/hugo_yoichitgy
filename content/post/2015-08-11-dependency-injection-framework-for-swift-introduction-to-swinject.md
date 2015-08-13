+++
date = "2015-08-11T13:20:41+09:00"
draft = false
slug = "dependency-injection-framework-for-swift-introduction-to-swinject"
tags = ["swinject", "dependency-injection", "swift"]
title = "Dependency Injection Framework for Swift - Introduction to Swinject"

+++

This blog post introduces [Swinject](https://github.com/Swinject/Swinject), a dependency injection framework for Swift. Swift 2 will come with [protocol extension](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID521) and encourage [protocol oriented programming](https://developer.apple.com/videos/wwdc/2015/?id=408). In addition, Xcode 7 will introduce [UI testing](https://developer.apple.com/videos/wwdc/2015/?id=406). In this context, it is getting more important to decouple components of an app by protocols. The typical pattern of the decoupling is called [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection).

## Dependency Injection

Let's start with an example. You are going to develop an app that lists current weather of some locations like the screenshot image below. The weather information will be received from a server through an API, and the data are used to present in the table view. Of course you will write unit tests. According to the screenshot, the tests will expect that the weather in Montreal[^1] should be "Clouds", in Moscow "Clear" and in Los Angeles "Clouds", but wait, if you write test code like that, will the tests pass tomorrow too? They will rarely pass because the weather changes.

The problem here is that the parts of network access and data processing are coupled. In other words, the data processing depends on the network access. If the dependency is hard-coded, it is difficult to write unit tests around the dependency. To solve the problem, the dependency should be passed from somewhere else. This is the dependency injection (DI) pattern. The external code provides dependencies to the client code. The injector is called DI container or simply container[^2].

![SwinjectSimpleExample Screenshot](/images/SwinjectSimpleExampleScreenshot.png)

## Swinject

[Swinject](https://github.com/Swinject/Swinject) is a lightweight dependency injection framework written in Swift to use with Swift. The framework APIs are easy to learn and use because of the generic type and first class function features of Swift. Swinject is available through [CocoaPods](https://cocoapods.org/) or [Carthage](https://github.com/Carthage/Carthage).

### Installation with CocoaPods

To install Swinject with CocoaPods, add the following lines to your `Podfile`[^3].

    source 'https://github.com/CocoaPods/Specs.git'
    platform :ios, '8.0'
    use_frameworks!

    pod 'Swinject', '~> 0.2.0'

Then run `pod install` command. For details of the installation and usage of CocoaPods, visit [its official website](https://cocoapods.org).

Later in the example app, we will use CocoaPods to install Swinject.

### Installation with Carthage

To install Swinject with Carthage, add the following line to your `Cartfile`[^4].

    github "Swinject/Swinject" ~> 0.2


Then run `carthage update` command. For details of the installation and usage of Carthage, visit [its project page](https://github.com/Carthage/Carthage).

## Basics

Before continuing to details of the example app, let me introduce the basics of dependency injection with Swinject. Its project has a playground and it is easy to try dependency injection with Swinject. Download [the source code](https://github.com/Swinject/Swinject/releases) or clone [the project](https://github.com/Swinject/Swinject), and open the project file to use the playground.

### Without Dependency Injection

Let's say we are writing a game to play with animals. First we will write the program without dependency injection. Here is `Cat` class to represent an animal,

    class Cat {
        let name: String

        init(name: String) {
            self.name = name
        }

        func sound() -> String {
            return "Meow!"
        }
    }

and `PetOwner` class has an instance of `Cat` as a pet to play with.

    class PetOwner {
        let pet = Cat(name: "Mimi")

        func play() -> String {
            return "I'm playing with \(pet.name). \(pet.sound())"
        }
    }

Now we can instantiate `PetOwner` to play.

    let petOwner = PetOwner()
    print(petOwner.play()) // prints "I'm playing with Mimi. Meow!"

This is great if everyone is a cat person, but in reality some are dog persons. Because the instantiation of a `Cat` is hard-coded, `PetOwner` class depends on `Cat` class. The dependency must be decoupled to support `Dog` or other classes.

### With Dependency Injection

Now is the time to start taking advantage of dependency injection. Here we are going to introduce `AnimalType` protocol to get rid of the dependency.

    protocol AnimalType {
        var name: String { get }
        func sound() -> String
    }

`Cat` class is modified to conform the protocol,

    class Cat: AnimalType {
        let name: String

        init(name: String) {
            self.name = name
        }

        func sound() -> String {
            return "Meow!"
        }
    }

and `PetOwner` class is modified to get an `AnimalType` injected through its initializer.

    class PetOwner {
        let pet: AnimalType

        init(pet: AnimalType) {
            self.pet = pet
        }

        func play() -> String {
            return "I'm playing with \(pet.name). \(pet.sound())"
        }
    }

Now we can inject the dependency to `AnimalType` protocol when a `PetOwner` instance is created.

    let catOwner = PetOwner(pet: Cat(name: "Mimi"))
    print(catOwner.play()) // prints "I'm playing with Mimi. Meow!"

If we have `Dog` class,

    class Dog: AnimalType {
        let name: String

        init(name: String) {
            self.name = name
        }

        func sound() -> String {
            return "Bow wow!"
        }
    }

we can play with a dog too.

    let dogOwner = PetOwner(pet: Dog(name: "Hachi"))
    print(dogOwner.play()) // prints "I'm playing with Hachi. Bow wow!"

So far, we injected the dependency of `PetOwner` by ourselves, but if we get more dependencies as the app evolved, it is harder to maintain dependency injection by hand. Let's introduce Swinject to manage the dependencies here.

To use Swinject, add the following line to a playground or source code.

    import Swinject

Then create an instance of `Container` and register the dependency.

    let container = Container()
    container.register(AnimalType.self) { _ in Cat(name: "Mimi") }
    container.register(PetOwner.self) { r in
        PetOwner(pet: r.resolve(AnimalType.self)!)
    }

In the code above, we told the `container` to resolve `AnimalType` to a `Cat` instance named "Mimi", and `PetOwner` to an instance with an `AnimalType` as a pet resolved by the `container`. The `resolve` method returns nil if the container cannot resolve an instance, but here we know `AnimalType` is already registered and force-unwrap the optional parameter.

We have got the configured container. Let's get an instance of `PetOwner` from the `container`.

    let petOwner = container.resolve(PetOwner.self)!
    print(petOwner.play()) // prints "I'm playing with Mimi. Meow!"

It is so simple to configure a `Container` and to retrieve a resolved instance with its dependencies injected.

## Conclusion

The concept of dependency injection has been introduced with the scenario to write unit tests for the weather app, and its basic use case has been demonstrated. With Swinject, it is easy to configure the dependencies and to get instances with the dependencies resolved. [In the next blog post](/post/dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-1/), we will see how to use Swinject with unit tests in the example weather app.

[^1]: The cities listed in the example app are the summer Olympic host cities since 1976.
[^2]: DI container is also called assembler, provider, builder, spring or injector.
[^3]: Specify version 0.1.0 if you use Xcode 6.4. Version 0.2 is required for Xcode 7 beta.
[^4]: Specify version 0.1 if you use Xcode 6.4. Version 0.2 is required for Xcode 7 beta.
