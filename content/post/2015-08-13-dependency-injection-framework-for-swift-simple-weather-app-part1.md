+++
date = "2015-08-13T09:16:58+09:00"
slug = "dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-1"
tags = ["swinject", "dependency-injection", "swift"]
title = "Dependency Injection Framework for Swift - Simple Weather App Example with Swinject Part 1/2"

+++

In [the last blog post](/post/dependency-injection-framework-for-swift-introduction-to-swinject/), we walked through the concept of dependency injection and basic usage of [Swinject](https://github.com/Swinject/Swinject), the dependency injection framework for Swift. In this blog post, we are going to develop the simple weather app that you saw its screenshot in the last blog post. During the simple but essential steps of the development, you will see how to get rid of tightly coupled dependencies by using the dependency injection pattern and Swinject.

The source code used in this blog post is available at [a repository on GitHub](https://github.com/Swinject/SwinjectSimpleExample).

![SwinjectSimpleExample Screenshot](/images/SwinjectSimpleExampleScreenshot.png)

## Requirements

- Xcode 7 (beta)
- OpenWeatherMap API key
- CocoaPods 0.38 or later

We are going to use Xcode 7 although it is still beta. It is beta 5 at the timing of writing this blog post. Xcode 7 supports `@testable import` to access `internal` types, functions or properties in unit test targets.

Also, we will use [OpenWeatherMap](http://openweathermap.org) for a free API to get weather information. [Sign up](http://home.openweathermap.org/users/sign_up) and get a free API key.

To install Swinject and some frameworks, we will use [CocoaPods](https://cocoapods.org).

## Preparation of the Project

Let's start with a new Xcode project. Select `File > New > Project...` menu and `iOS > Application > Single View Application` item. Set  its product name to `SwinjectSimpleExample`, language to Swift and devices to iPhone. Check `Include Unit Tests` only[^1], then save it anywhere in your local storage.

Then, we are going to install [Alamofire](https://github.com/Alamofire/Alamofire), [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON), [Swinject](https://github.com/Swinject/Swinject), [Quick](https://github.com/Quick/Quick) and [Nimble](https://github.com/Quick/Nimble) with [CocoaPods](https://cocoapods.org). Create `Podfile` with the following text content in the project root directory. Then run `pod install` command to install them. Since Xcode 7 is still beta, specific commits of Alamofire and SwiftyJSON are specified[^2].

    source 'https://github.com/CocoaPods/Specs.git'
    platform :ios, '8.0'
    use_frameworks!

    pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :commit => '1b7b1f1aa'
    pod 'SwiftyJSON', :git => 'https://github.com/SwiftyJSON/SwiftyJSON.git', :commit => '45ca854ce'
    pod 'Swinject', '~> 0.2'

    target 'SwinjectSimpleExampleTests' do
        pod 'Quick', '~> 0.5.0'
        pod 'Nimble', '2.0.0-rc.2'
    end

Alamofire is a networking library to write request and asynchronous response simply. SwiftyJSON is a library to access JSON elements simply. Quick is a [behavior-driven development](https://en.wikipedia.org/wiki/Behavior-driven_development) framework to write tests as specs in simple structures. Nimble is a matcher framework that is expressive and supports asynchronous tests. For details, please visit their project pages.

To use the free weather API on iOS 9, we have to allow HTTP connections. Open `Info.plist` and add `NSAppTransportSecurity` dictionary with `NSAllowsArbitraryLoads` element set to `true`[^3].  [Here](http://stackoverflow.com/questions/30720813/cfnetwork-sslhandshake-failed-ios-9) is more information about the setting and its background.

## Without Dependency Injection

First, without dependency injection, we are going to implement a model to handle weather information retrieved through the network service. We will see what can be a problem if we do not care about coupled dependencies.

Add `City.swift` to `SwinjectSimpleExample` group in the project. We define `City` to be an entity representing a city with weather information.

**City.swift**

    struct City {
        let id: Int
        let name: String
        let weather: String
    }

Add `OpenWeatherMap.swift` to store configurations of OpenWeatherMap API. Here please fill `apiKey` with your own API key.

**OpenWeatherMap.swift**

    struct OpenWeatherMap {
        private static let apiKey = "YOUR API KEY HERE"

        private static let cityIds = [
            6077243, 524901, 5368361, 1835848, 3128760, 4180439,
            2147714, 264371, 1816670, 2643743, 3451190, 1850147
        ]

        static let url = "http://api.openweathermap.org/data/2.5/group"

        static var parameters: [String: String] {
            return [
                "APPID": apiKey,
                "id": ",".join(cityIds.map { String($0) })
            ]
        }
    }

Add `WeatherFetcher.swift` to implement `WeatherFetcher`, which has `fetch` function taking a callback to handle an optional array of `Cities` returned from OpenWeatherMap.

**WeatherFetcher.swift**

    import Foundation
    import Alamofire
    import SwiftyJSON

    struct WeatherFetcher {
        static func fetch(response: [City]? -> ()) {
            Alamofire.request(.GET, OpenWeatherMap.url, parameters: OpenWeatherMap.parameters)
                .response { _, _, data, _ in
                    let cities = data.map { decode($0) }
                    response(cities)
                }
        }

        private static func decode(data: NSData) -> [City] {
            let json = JSON(data: data)
            var cities = [City]()
            for (_, j) in json["list"] {
                if let id = j["id"].int {
                    let city = City(
                        id: id,
                        name: j["name"].string ?? "",
                        weather: j["weather"][0]["main"].string ?? "")
                    cities.append(city)
                }
            }
            return cities
        }
    }

The `fetch` function uses Alamofire to send a request to the server and to get a response as JSON data asynchronously. The specifications of API call and response JSON format are described in ["Call for several city IDs" section of OpenWeatherMap site](http://openweathermap.org/current#severalid).
The `data` parameter in the closure passed to `response` from `Alamofire` is `nil` if the response has an error. We do not care about details of the error and just pass `nil` to the callback to `fetch` in this example although the error should be handled in a product app.

The `decode` function parses the JSON data returned from the server. It is called as `data.map { decode($0) }` in `fetch` where `map` executes the trailing closure if `data` is not `nil`, otherwise returns nil. The `decode` function uses SwiftyJSON to map the JSON data to an array of our `City` entities.

Let's add a unit test to `SwinjectSimpleExampleTests` group in our project. The filename is `WeatherFetcherSpec.swift` and its target is set to `SwinjectSimpleExampleTests` when we create the file. The test is going to check whether the weather data can be retrieved and parsed correctly.

**WeatherFetcherSpec.swift**

    import Quick
    import Nimble
    @testable import SwinjectSimpleExample

    class WeatherFetcherSpec: QuickSpec {
        override func spec() {
            it("returns cities.") {
                var cities: [City]?
                WeatherFetcher.fetch { cities = $0 }

                expect(cities).toEventuallyNot(beNil())
                expect(cities?.count).toEventually(equal(12))
                expect(cities?[0].id).toEventually(equal(6077243))
                expect(cities?[0].name).toEventually(equal("Montreal"))
                expect(cities?[0].weather).toEventually(equal("Clouds"))
            }
        }
    }

With Quick and Nimble, each test is written in an `it` closure, and each expectation is expressed as `expect(something).to(condition)` or `expect(something).toNot(condition)` synchronously, or `expect(something).toEventually(condition)` or `expect(something).toEventuallyNot(condition)` asynchronously. `WeatherFetcher.fetch` sets `cities` asynchronously when weather data is retrieved, so we use the latter ones here.

First, we check `cities`, which is initialized with `nil`, should be set to an array after `fetch` invokes the callback asynchronously. Second, the number of `cities` should be `12` because our request to the API has 12 city IDs. From the third to fifth, we check only the first city for simplicity. The `id`, `name` and `weather` should be `6077243`, `"Montreal"` and `"Clouds"` respectively.

Okay. We are ready to run the unit test. Type `Command-U` to run. Did you see the test passed? I think some people saw it passed, but the others not. Why? Because the weather in "Montreal" in the real world right now must be "Clouds" to pass the test. How can we write a test passing regardless of the current weather? It is actually difficult to write if the part parsing JSON data depends on the part retrieving the data from the server.

## With Dependency Injection

In the last section, we found the tightly coupled dependency of the parser on the network, namely Alamofire, made the test difficult. In this section, we are going to decouple them, inject the dependency and write a better test.

First, add `Networking.swift` with the following protocol definition. It has `request` method taking a callback to pass response data from the network.

**Networking.swift**

    import Foundation

    protocol Networking {
        func request(response: NSData? -> ())
    }

Add `Network.swift` to implement `Network` that conforms `Networking` protocol. It encapsulates Alamofire.

**Network.swift**

    import Foundation
    import Alamofire

    struct Network : Networking {
        func request(response: NSData? -> ()) {
            Alamofire.request(.GET, OpenWeatherMap.url, parameters: OpenWeatherMap.parameters)
                .response { _, _, data, _ in
                    response(data)
                }
        }
    }

Modify `WeatherFetcher` to get `Networking` injected when it is instantiated and to use it to request weather data to the server. Note that `fetch` and `decode` functions were `static` in the last section, but here they are instance methods to use the `networking` property. A default initializer taking `networking` is implicitly created by Swift. Now `WeatherFetcher` has no dependency on Alamofire.

**WeatherFetcher.swift**

    struct WeatherFetcher {
        let networking: Networking

        func fetch(response: [City]? -> ()) {
            networking.request { data in
                let cities = data.map { self.decode($0) }
                response(cities)
            }
        }

        private func decode(data: NSData) -> [City] {
            let json = JSON(data: data)
            var cities = [City]()
            for (_, j) in json["list"] {
                if let id = j["id"].int {
                    let city = City(
                        id: id,
                        name: j["name"].string ?? "",
                        weather: j["weather"][0]["main"].string ?? "")
                    cities.append(city)
                }
            }
            return cities
        }
    }

Then modify `WeatherFetcherSpec` to test the decoupled network and JSON parser.

**WeatherFetcherSpec.swift**

    import Quick
    import Nimble
    import Swinject
    @testable import SwinjectSimpleExample

    class WeatherFetcherSpec: QuickSpec {
        struct StubNetwork: Networking {
            private static let json =
            "{" +
                "\"list\": [" +
                    "{" +
                        "\"id\": 2643743," +
                        "\"name\": \"London\"," +
                        "\"weather\": [" +
                            "{" +
                                "\"main\": \"Rain\"" +
                            "}" +
                        "]" +
                    "}," +
                    "{" +
                        "\"id\": 3451190," +
                        "\"name\": \"Rio de Janeiro\"," +
                        "\"weather\": [" +
                            "{" +
                                "\"main\": \"Clear\"" +
                            "}" +
                        "]" +
                    "}" +
                "]" +
            "}"

            func request(response: NSData? -> ()) {
                let data = StubNetwork.json.dataUsingEncoding(
                    NSUTF8StringEncoding, allowLossyConversion: false)
                response(data)
            }
        }

        override func spec() {
            var container: Container!
            beforeEach {
                container = Container()

                // Registrations for the network using Alamofire.
                container.register(Networking.self) { _ in Network() }
                container.register(WeatherFetcher.self) { r in
                    WeatherFetcher(networking: r.resolve(Networking.self)!)
                }

                // Registration for the stub network.
                container.register(Networking.self, name: "stub") { _ in
                    StubNetwork()
                }
                container.register(WeatherFetcher.self, name: "stub") { r in
                    WeatherFetcher(
                        networking: r.resolve(Networking.self, name: "stub")!)
                }
            }

            it("returns cities.") {
                var cities: [City]?
                let fetcher = container.resolve(WeatherFetcher.self)!
                fetcher.fetch { cities = $0 }

                expect(cities).toEventuallyNot(beNil())
                expect(cities?.count).toEventually(beGreaterThan(0))
            }
            it("fills weather data.") {
                var cities: [City]?
                let fetcher = container.resolve(WeatherFetcher.self, name: "stub")!
                fetcher.fetch { cities = $0 }

                expect(cities?[0].id).toEventually(equal(2643743))
                expect(cities?[0].name).toEventually(equal("London"))
                expect(cities?[0].weather).toEventually(equal("Rain"))
                expect(cities?[1].id).toEventually(equal(3451190))
                expect(cities?[1].name).toEventually(equal("Rio de Janeiro"))
                expect(cities?[1].weather).toEventually(equal("Clear"))
            }
        }
    }

`StubNetwork` is a stub that conforms `Networking`. It has a definition of JSON data that has the same structure as the data returned from the server. Its `request` method returns the identical data any time regardless of the current weather in the real world. In `spec`, `container` is configured at the beginning, and it is used later in the two `it` specifications. Without a registration name, `container` is configured to use `Network`. With the registration name "stub", it is configured to use `StubNetwork`.

The first `it` tests that the real network through Alamofire returns some JSON data[^4] by getting an instance of `WeatherFetcher` from `container` without a registration name. We do not test detail of `cities`. We just confirm that `fetch` can get some data from the server.

The second `it` tests that the JSON data are parsed correctly by getting an instance of `WeatherFetcher` with the registration name "stub". Because the stub returns two cities as defined in `StubNetwork`, we write expectations for the two cities and check whether each expectation asynchronously gets the value specified in the stub definition.

Okay. We are ready to run the tests. Type `Command-U` to run. This time you got the tests passed regardless of the current weather, didn't you? This is the advantage of the dependency injection pattern to decouple a component from another, in this example decoupling of the parser component from network component.

## Conclusion

The problem of dependencies to write unit tests has been explained and fixed with dependency injection in the scenario to develop the app using the network service and JSON parser. By decoupling these two parts, the unit tests have become reproducible under any circumstances. In the next blog post, we will develop the UI part of the example app to learn how to use Swinject in a product app.

[^1]: UI tests are excluded because still Xcode 7 is beta (just caring NDA). This blog post will be updated to include them after Xcode 7 is officially released.
[^2]: `Podfile` in this blog post will be updated after Xcode 7 is officially released.
[^3]: Actually this setting is not preferable if you develop an app to release. In this blog post, I used the setting just because the free API only supports HTTP.
[^4]: This test may fail if the network is disconnected or has a problem, but these cases can be practically ignored in our unit tests.
