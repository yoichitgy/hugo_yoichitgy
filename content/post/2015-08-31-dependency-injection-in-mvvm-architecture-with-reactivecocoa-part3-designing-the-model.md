+++
date = "2015-08-31T01:10:55+09:00"
slug = "dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-3-designing-the-model"
tags = ["swinject", "dependency-injection", "mvvm", "reactivecocoa", "swift", "alamofire", "himotoki"]
title = "Dependency Injection in MVVM Architecture with ReactiveCocoa Part 3: Designing the Model"

+++

- **Updated on Nov 20, 2015** to migrate to ReactiveCocoa v4.0.0 alpha 3, Alamofire v3.x and Himotoki v1.3.
- **Updated on Oct 1, 2015** for the release versions of Swift 2 and Xcode 7.

In [the last blog post](/post/dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-2-project-setup/), we setup an Xcode project to develop an app composed of Model, View and ViewModel frameworks. In this blog post, we are going to develop the Model part in the MVVM architecture. We will learn how to design our Model consisting of entities and services with dependencies injected. Decoupling of the dependencies is the advantage of the MVVM architecture. [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) is used for event handling, which is essential to decouple Model, View and ViewModel in the MVVM architecture. We will also learn how to use [Himotoki](https://github.com/ikesyo/Himotoki) to define mappings from JSON data to Swift types.

The source code used in the blog posts is available at:

- [SwinjectMVVMExample](https://github.com/Swinject/SwinjectMVVMExample): Complete version of the project.
- [SwinjectMVVMExample_ForBlog](https://github.com/yoichitgy/SwinjectMVVMExample_ForBlog): Simplified version of the project to follow the blog posts (except updates of Xcode and frameworks).

![SwinjectMVVMExample ScreenRecord](/images/post/2015-08/SwinjectMVVMExampleScreenRecord.gif)

## Notice

To use with Swift 2 and Xcode 7, ReactiveCocoa [version 4.0](https://github.com/ReactiveCocoa/ReactiveCocoa/releases) is used though it is still in an alpha version at the moment. Notice that ReactiveCocoa 3.0 has functional style APIs like `|> map` or `|> flatMap`, but version 4 APIs are in protocol oriented and fluent style like `.map()` or `.flatMap()`.

## Himotoki

[Himotoki](https://github.com/ikesyo/Himotoki) is a type-safe JSON decoding library for Swift. It maps JSON elements to properties of a type with a simple definition of the mappings as `Decodable` protocol. The advantage of Himotoki is the support of mappings to immutable (`let`) properties.

Its usage is simple. Assume that you want to map JSON like `{ "some_name": "Himotoki", "some_value": 1 }` to `SomeValue` type. To define the mappings, make `SomeValue` type conform `Decodable` protocol.

    struct SomeValue {
        let name: String
        let value: Int
    }

    extension SomeValue: Decodable {
        static func decode(e: Extractor) throws -> Group {
            return try SomeValue(
                name: e <| "some_name",
                value: e <| "some_value"
            )
        }
    }

In `decode` function, the mappings are defined as the parameters to `build` function. Here `some_name` is mapped to `name` property of `SomeValue`, and `some_value` to `value` property. The mappings are defined in the order as the properties of `SomeValue` type are defined.

To get an instance mapped from JSON data, call `decode` function with JSON data as `[String: AnyObject]` returned from Alamofire or NSJSONSerialization.

    func testSomeValue() {
        // JSON data returned from Alamofire or NSJSONSerialization.
        let json: [String: AnyObject] = ["some_name": "Himotoki", "some_value": 1]

        let v: SomeValue? = try? decode(json)
        XCTAssert(v != nil)
        XCTAssert(v?.name == "Himotoki")
        XCTAssert(v?.value == 1)
    }

With Himotoki, you can reduce the code to handle JSON data[^1]. It also supports nested data, optional parameters and more. Refer to [the project page](https://github.com/ikesyo/Himotoki) for the details.

## Pixabay API Spec

According to [the API documentation of Pixabay](https://pixabay.com/api/docs/), a JSON response from Pixabay server is in the following format. It contains an array of image information (`hits`), and the numbers of total images (`total`) and available images (`totalHits`).

    {
        "total": 12274,
        "totalHits": 240,
        "hits": [
            {
                "id": 11574,
                "pageURL": "https://pixabay.com/en/sonnenblumen-sonnenblumenfeld-flora-11574/",
                "type": "photo",
                "tags": "sunflower, sunflower field, flora",
                "previewURL": "https://pixabay.com/static/uploads/photo/2012/01/07/21/56/sunflower-11574_150.jpg",
                "previewWidth": 150,
                "previewHeight": 92,
                "webformatURL": "https://pixabay.com/get/3b4f5d71752e6ce9cbcf/1356479243/aca42219d23fd9fe0cc6f1cc_640.jpg",
                "webformatWidth": 640,
                "webformatHeight": 396,
                "imageWidth": 1280,
                "imageHeight": 792,
                "views": 10928,
                "downloads": 1649,
                "likes": 70,
                "user": "WikiImages"
            },
            {
                "id": "256",
                "pageURL": "https://pixabay.com/en/example-image-256/",
                "type": "photo",
                // ... etc.
            },
            //... 18 more hits for page number 1
        ]
    }

## Model Design Overview

We are going to design our Model to be composed of entities and services. In short, an entity is a concept or object that exists in a model[^2]. A service is a stateless operation that does not fit in an entity.

To decouple ViewModel and Model, and Model and external system, the interfaces are defined by protocols. In the diagram below, `ImageSearching` and `Networking` are protocols. `ImageSearch` and `Network` are their implementations conforming the protocols. `ViewModel` accesses `Model` through `ImageSearching` protocol, and its implementation `ImageSearch` accesses the external system through `Networking` protocol. The external system raises events with JSON data. The data are converted to `ResponseEntity` and `ImageEntity` by `ImageSearch` when the events are propagated to `ViewModel`.

![Model Design](/images/post/2015-08/SwinjectMVVMExampleModelDesign.png)

## Entities

In this section, we are going to define the entities representing an image and response obtained from Pixabay. Add `ImageEntity.swift` with the following content to `ExampleModel` target. To make sure the file is added to the target, right click on `ExampleModel` group (folder icon) in Project Navigator, and choose `New File...` then `Swift File`. When Xcode asks targets to add the file to, check only `ExampleModel`.

**ImageEntity.swift**

    import Himotoki

    public struct ImageEntity {
        public let id: UInt64

        public let pageURL: String
        public let pageImageWidth: Int
        public let pageImageHeight: Int

        public let previewURL: String
        public let previewWidth: Int
        public let previewHeight: Int

        public let imageURL: String
        public let imageWidth: Int
        public let imageHeight: Int

        public let viewCount: Int64
        public let downloadCount: Int64
        public let likeCount: Int64
        public let tags: [String]
        public let username: String
    }

    // MARK: Decodable
    extension ImageEntity: Decodable {
        public static func decode(e: Extractor) throws -> ImageEntity {
            let splitCSV: String -> [String] = { csv in
                csv.characters
                    .split { $0 == "," }
                    .map {
                        String($0).stringByTrimmingCharactersInSet(
                            NSCharacterSet.whitespaceCharacterSet())
                    }
            }

            return try ImageEntity(
                id: e <| "id",

                pageURL: e <| "pageURL",
                pageImageWidth: e <| "imageWidth",
                pageImageHeight: e <| "imageHeight",

                previewURL: e <| "previewURL",
                previewWidth: e <| "previewWidth",
                previewHeight: e <| "previewHeight",

                imageURL: e <| "webformatURL",
                imageWidth: e <| "webformatWidth",
                imageHeight: e <| "webformatHeight",

                viewCount: e <| "views",
                downloadCount: e <| "downloads",
                likeCount: e <| "likes",
                tags: (try? e <| "tags").map(splitCSV) ?? [],
                username: e <| "user"
            )
        }
    }

Notice that `ImageEntity` is defined as a `struct` with its properties immutable. The immutability keeps the users of the entity safe[^3]. Accessibility of the type is `public` because it is referenced from `ExampleViewModel` target.

Some properties of `ImageEntity` are named differently from the JSON elements to match Swift naming convention and our app. The properties `id`, `viewCount`, `downloadCount` and `likeCount` are declared as `UInt64` or `Int64` to ensure they accept large values even in 32-bit system. JSON element `tags` as a [CSV](https://en.wikipedia.org/wiki/Comma-separated_values) format string is mapped to an array of split strings by `(try? e <| "tags").map(splitCSV)`. By applying `?? []` to the returned value, an empty array is assigned instead of `nil` if the `tags` JSON element is missing.

Add `ResponseEntity.swift` to `ExampleModel` target with the following text contents. Here `<||` operator is used to map an array. The `total` element in the JSON is ignored. If we find it is necessary, we can add it later.

**ResponseEntity.swift**

    import Himotoki

    public struct ResponseEntity {
        public let totalCount: Int64
        public let images: [ImageEntity]
    }

    // MARK: Decodable
    extension ResponseEntity: Decodable {
        public static func decode(e: Extractor) throws -> ResponseEntity {
            return try ResponseEntity(
                totalCount: e <| "totalHits",
                images: e <|| "hits"
            )
        }
    }

To test `ImageEntity`, add `Dummy.swift` and `ImageEntitySpec.swift` with the following contents to `ExampleModelTests` target.

**Dummy.swift**

    let imageJSON: [String: AnyObject] = [
        "id": 12345,
        "pageURL": "https://somewhere.com/page/",
        "imageWidth": 2000,
        "imageHeight": 1000,
        "previewURL": "https://somewhere.com/preview.jpg",
        "previewWidth": 200,
        "previewHeight": 100,
        "webformatURL": "https://somewhere.com/image.jpg",
        "webformatWidth": 600,
        "webformatHeight": 300,
        "views": 54321,
        "downloads": 4321,
        "likes": 321,
        "tags": "a, b c, d ",
        "user": "Swinject"
    ]

**ImageEntitySpec.swift**

    import Quick
    import Nimble
    import Himotoki
    @testable import ExampleModel

    class ImageEntitySpec: QuickSpec {
        override func spec() {
            it("parses JSON data to create a new instance.") {
                let image: ImageEntity? = try? decode(imageJSON)

                expect(image).notTo(beNil())
                expect(image?.id) == 12345
                expect(image?.pageURL) == "https://somewhere.com/page/"
                expect(image?.pageImageWidth) == 2000
                expect(image?.pageImageHeight) == 1000
                expect(image?.previewURL) == "https://somewhere.com/preview.jpg"
                expect(image?.previewWidth) == 200
                expect(image?.previewHeight) == 100
                expect(image?.imageURL) == "https://somewhere.com/image.jpg"
                expect(image?.imageWidth) == 600
                expect(image?.imageHeight) == 300
                expect(image?.viewCount) == 54321
                expect(image?.downloadCount) == 4321
                expect(image?.likeCount) == 321
                expect(image?.tags) == ["a", "b c", "d"]
                expect(image?.username) == "Swinject"
            }
            it("gets an empty array if tags element is nil.") {
                var missingJSON = imageJSON
                missingJSON["tags"] = nil
                let image: ImageEntity? = try? decode(missingJSON)

                expect(image).notTo(beNil())
                expect(image?.tags.isEmpty).to(beTrue())
            }
            it("throws an error if any of JSON elements except tags is missing.") {
                for key in imageJSON.keys where key != "tags" {
                    var missingJSON = imageJSON
                    missingJSON[key] = nil
                    let image: ImageEntity? = try? decode(missingJSON)

                    expect(image).to(beNil())
                }
            }
            it("ignores an extra JOSN element.") {
                var extraJSON = imageJSON
                extraJSON["extraKey"] = "extra element"
                let image: ImageEntity? = try? decode(extraJSON)

                expect(image).notTo(beNil())
            }
        }
    }

In `Dummy.swift`, a dummy instance of JSON data is defined as `imageJSON`. Since Himotoki handles JSON data as `[String: AnyObject]` type returned from Alamofire or NSJSONSerialization, `imageJSON` is defined as `[String: AnyObject]`, not as `String`.

`it("parses JSON data to create a new instance.")` checks that all the properties of `ImageEntity` are mapped from the JSON data.

`it("gets an empty array if tags element is nil.")` checks that `tags` property is set to an empty array if `tags` element is missing in the JSON data.

`it("throws an error if any of JSON elements except tags is missing.")` checks that `decode` function throws an error if any of the JSON elements except `tags` is missing. Incorrect or broken JSON data can be detected to see whether the returned value from `try?` is `nil`.

`it("ignores an extra JOSN element.")` checks that `decode` function returns an `ImageEntity` instance even if an extra JSON element exists. Ignoring extra elements, our JSON mapping is stable for future changes on the Pixabay API.

Next add `ResponseEntitySpec.swift` with the following content to `ExampleModelTests` target. The test is simple as you see.

**ResponseEntitySpec.swift**

    import Quick
    import Nimble
    import Himotoki
    @testable import ExampleModel

    class ResponseEntitySpec: QuickSpec {
        override func spec() {
            let json: [String: AnyObject] = [
                "totalHits": 123,
                "hits": [imageJSON, imageJSON]
            ]

            it("parses JSON data to create a new instance.") {
                let response: ResponseEntity? = try? decode(json)

                expect(response).notTo(beNil())
                expect(response?.totalCount) == 123
                expect(response?.images.count) == 2
            }
        }
    }

Type `Command-U` to run the unit tests. Passed. The tests passed without any problems, but, in actual development, we iterate fixing the entities and tests until the tests pass.

## Network Service

As we did in [the simple weather app example](/post/dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-1/), we are going to encapsulate Alamofire within `Network` type conforming `Networking` protocol. This time, we use ReactiveCocoa to handle the network event. Add `Networking.swift` and `Network.swift` with the following contents to `ExampleModel` target.

**Networking.swift**

    import ReactiveCocoa

    public protocol Networking {
        func requestJSON(url: String, parameters: [String : AnyObject]?)
            -> SignalProducer<AnyObject, NetworkError>
    }

**Network.swift**

    import ReactiveCocoa
    import Alamofire

    public final class Network: Networking {
        private let queue = dispatch_queue_create(
            "SwinjectMMVMExample.ExampleModel.Network.Queue",
            DISPATCH_QUEUE_SERIAL)

        public init() { }

        public func requestJSON(url: String, parameters: [String : AnyObject]?)
            -> SignalProducer<AnyObject, NetworkError>
        {
            return SignalProducer { observer, disposable in
                let serializer = Alamofire.Request.JSONResponseSerializer()
                Alamofire.request(.GET, url, parameters: parameters)
                    .response(queue: self.queue, responseSerializer: serializer) {
                        response in
                        switch response.result {
                        case .Success(let value):
                            observer.sendNext(value)
                            observer.sendCompleted()
                        case .Failure(let error):
                            observer.sendFailed(NetworkError(error: error))
                        }
                    }
            }
        }
    }

In [the previous example](/post/dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-1/), the network service took a callback to pass a response. This time, `requestJSON` method returns `SignalProducer<AnyObject, NetworkError>` to deliver network events to its observer. The `SignalProducer` takes a generic argument as `AnyObject` because Alamofire (or NSJSONSerialization) returns JSON data as `AnyObject` that is actually an array or dictionary.

The `requestJSON` method creates `SignalProducer` instance with its trailing closure, and returns the instance. Within the closure, a response event of Alamofire is converted to a ReactiveCocoa event by calling `sendNext`, `sendCompleted` and `sendError`. Notice that the closure passed to `SignalProducer` is not actually invoked until `start` method is called on the `SignalProducer` instance.

A dispatch queue is passed to Alamofire to run the response in a background thread because Alamofire by default runs the response in the main thread (main queue).

`sendError` function takes an instance of `NetworkError` type. Add `NetworkError.swift` with the following content to `ExampleModel` target. It converts an `NSError` passed from Alamofire to our own error type[^4].

**NetworkError.swift**

    import Foundation

    public enum NetworkError: ErrorType {
        /// Unknown or not supported error.
        case Unknown

        /// Not connected to the internet.
        case NotConnectedToInternet

        /// International data roaming turned off.
        case InternationalRoamingOff

        /// Cannot reach the server.
        case NotReachedServer

        /// Connection is lost.
        case ConnectionLost

        /// Incorrect data returned from the server.
        case IncorrectDataReturned

        internal init(error: NSError) {
            if error.domain == NSURLErrorDomain {
                switch error.code {
                case NSURLErrorUnknown:
                    self = .Unknown
                case NSURLErrorCancelled:
                    self = .Unknown // Cancellation is not used in this project.
                case NSURLErrorBadURL:
                    self = .IncorrectDataReturned // Because it is caused by a bad URL returned in a JSON response from the server.
                case NSURLErrorTimedOut:
                    self = .NotReachedServer
                case NSURLErrorUnsupportedURL:
                    self = .IncorrectDataReturned
                case NSURLErrorCannotFindHost, NSURLErrorCannotConnectToHost:
                    self = .NotReachedServer
                case NSURLErrorDataLengthExceedsMaximum:
                    self = .IncorrectDataReturned
                case NSURLErrorNetworkConnectionLost:
                    self = .ConnectionLost
                case NSURLErrorDNSLookupFailed:
                    self = .NotReachedServer
                case NSURLErrorHTTPTooManyRedirects:
                    self = .Unknown
                case NSURLErrorResourceUnavailable:
                    self = .IncorrectDataReturned
                case NSURLErrorNotConnectedToInternet:
                    self = .NotConnectedToInternet
                case NSURLErrorRedirectToNonExistentLocation, NSURLErrorBadServerResponse:
                    self = .IncorrectDataReturned
                case NSURLErrorUserCancelledAuthentication, NSURLErrorUserAuthenticationRequired:
                    self = .Unknown
                case NSURLErrorZeroByteResource, NSURLErrorCannotDecodeRawData, NSURLErrorCannotDecodeContentData:
                    self = .IncorrectDataReturned
                case NSURLErrorCannotParseResponse:
                    self = .IncorrectDataReturned
                case NSURLErrorInternationalRoamingOff:
                    self = .InternationalRoamingOff
                case NSURLErrorCallIsActive, NSURLErrorDataNotAllowed, NSURLErrorRequestBodyStreamExhausted:
                    self = .Unknown
                case NSURLErrorFileDoesNotExist, NSURLErrorFileIsDirectory:
                    self = .IncorrectDataReturned
                case
                NSURLErrorNoPermissionsToReadFile,
                NSURLErrorSecureConnectionFailed,
                NSURLErrorServerCertificateHasBadDate,
                NSURLErrorServerCertificateUntrusted,
                NSURLErrorServerCertificateHasUnknownRoot,
                NSURLErrorServerCertificateNotYetValid,
                NSURLErrorClientCertificateRejected,
                NSURLErrorClientCertificateRequired,
                NSURLErrorCannotLoadFromNetwork,
                NSURLErrorCannotCreateFile,
                NSURLErrorCannotOpenFile,
                NSURLErrorCannotCloseFile,
                NSURLErrorCannotWriteToFile,
                NSURLErrorCannotRemoveFile,
                NSURLErrorCannotMoveFile,
                NSURLErrorDownloadDecodingFailedMidStream,
                NSURLErrorDownloadDecodingFailedToComplete:
                    self = .Unknown
                default:
                    self = .Unknown
                }
            }
            else {
                self = .Unknown
            }
        }
    }

Let's add unit tests for the `Network` type. Add `NetworkSpec.swift` with the following content to `ExampleModelTests` target.

**NetworkSpec.swift**

    import Quick
    import Nimble
    @testable import ExampleModel

    class NetworkSpec: QuickSpec {
        override func spec() {
            var network: Network!
            beforeEach {
                network = Network()
            }

            describe("JSON") {
                it("eventually gets JSON data as specified with parameters.") {
                    var json: [String: AnyObject]? = nil
                    let url = "https://httpbin.org/get"
                    network.requestJSON(url, parameters: ["a": "b", "x": "y"])
                        .on(next: { json = $0 as? [String: AnyObject] })
                        .start()

                    expect(json).toEventuallyNot(beNil(), timeout: 5)
                    expect((json?["args"] as? [String: AnyObject])?["a"] as? String)
                        .toEventually(equal("b"), timeout: 5)
                    expect((json?["args"] as? [String: AnyObject])?["x"] as? String)
                        .toEventually(equal("y"), timeout: 5)
                }
                it("eventually gets an error if the network has a problem.") {
                    var error: NetworkError? = nil
                    let url = "https://not.existing.server.comm/get"
                    network.requestJSON(url, parameters: ["a": "b", "x": "y"])
                        .on(error: { error = $0 })
                        .start()

                    expect(error)
                        .toEventually(equal(NetworkError.NotReachedServer), timeout: 5)
                }
            }
        }
    }

Here [httpbin.org](https://httpbin.org/) is used as a stable and simple server to test the network. It is used by [the unit tests of Alamofire](https://github.com/Alamofire/Alamofire/blob/1978c2c926b0eabedc858d4cde0533e00686ccd6/Tests/ResponseTests.swift) too. Access [https://httpbin.org/get?a=b&x=y](https://httpbin.org/get?a=b&x=y) with a browser, and you will see how it works.

The first test checks the JSON response has `"a"` and `"x"` elements with `"b"` and `"y"` values as specified in the request parameters. `json` parameter is used to store the response from the server asynchronously. To add an observer (namely an event handler) to the `SignalProducer` returned from `requestJSON`, `on` method is used with a closure to set the `json` response. Then `start` is called to get the `SignalProducer` initiating its signal. The response is checked asynchronously with `toEventually` method.

The second test checks `NetworkError` is sent in case of an error. To emulate an error, a URL that does not exist is passed to `requestJSON`.

Run the unit tests, and let's move on to the next section.

## Image Search Service

In this section, we are going to define the service to search images through Pixabay API. This is the main part of `ExampleModel`.

First, add `Pixabay.swift` with the following content to `ExampleModel` target. It defines the URL and parameters for the API. Please fill `apiUsername` and `apiKey` with [your own username and API key obtained from Pixabay](https://pixabay.com/api/docs/).

**Pixabay.swift**

    internal struct Pixabay {
        internal static let apiURL = "https://pixabay.com/api/"

        internal static var requestParameters: [String: AnyObject] {
            return [
                "username": Config.apiUsername,
                "key": Config.apiKey,
                "image_type": "photo",
                "safesearch": true,
                "per_page": 50,
            ]
        }
    }

    extension Pixabay {
        private struct Config {
            private static let apiUsername = "" // Fill with your own username.
            private static let apiKey = "" // Fill with your own API key.
        }
    }

Second, add `ImageSearching.swift` with the following content to `ExampleModel` target. The protocol has `searchImages` method returning `SignalProducer` of `ResponseEntity`.

**ImageSearching.swift**

    import ReactiveCocoa

    public protocol ImageSearching {
        func searchImages() -> SignalProducer<ResponseEntity, NetworkError>
    }

Third, add `ImageSearch.swift` with the following content to `ExampleModel` target.

**ImageSearch.swift**

    import ReactiveCocoa
    import Result
    import Himotoki

    public final class ImageSearch: ImageSearching {
        private let network: Networking

        public init(network: Networking) {
            self.network = network
        }

        public func searchImages() -> SignalProducer<ResponseEntity, NetworkError> {
            let url = Pixabay.apiURL
            let parameters = Pixabay.requestParameters
            return network.requestJSON(url, parameters: parameters)
                .attemptMap { json in
                    if let response = (try? decode(json)) as ResponseEntity? {
                        return Result(value: response)
                    }
                    else {
                        return Result(error: .IncorrectDataReturned)
                    }
            }
        }
    }

`ImageSearch` has dependency on `Networking` injected through the initializer as [initializer injection pattern](https://github.com/Swinject/Swinject/blob/master/Documentation/InjectionPatterns.md).

`searchImages` method converts `SignalProducer<AnyObject, NetworkError>` returned from `network.requestJSON` to `SignalProducer<ResponseEntity, NetworkError>`. To convert the `SignalProducer`, `attemptMap` is used. The closure passed to `attemptMap` calls `decode` to map the JSON data to a `ResponseEntity` instance. If the mapping succeeds, the mapped response is returned as `Result(value: response)`. Otherwise[^5], an error is returned as `Result(error: .IncorrectDataReturned)`. If you convert a value not to an error but to another value only, you can just use `map` method on `SignalProducer`.

The cast of `(try? decode(json)) as ResponseEntity?` looks irregular, but it helps Swift compiler infer that `decode` function on `ResponseEntity` type should be used. If the cast is `(try? decode(json)) as? ResponseEntity`, the source code cannot be compiled.

Let's write unit tests. Add `ImageSearchSpec.swift` with the following content to `ExampleModelTests` target.

**ImageSearchSpec.swift**

    import Quick
    import Nimble
    import ReactiveCocoa
    @testable import ExampleModel

    class ImageSearchSpec: QuickSpec {
        // MARK: Stub
        class GoodStubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                var imageJSON0 = imageJSON
                imageJSON0["id"] = 0
                var imageJSON1 = imageJSON
                imageJSON1["id"] = 1
                let json: [String: AnyObject] = [
                    "totalHits": 123,
                    "hits": [imageJSON0, imageJSON1]
                ]

                return SignalProducer { observer, disposable in
                    sendNext(observer, json)
                    sendCompleted(observer)
                }.observeOn(QueueScheduler())
            }
        }

        class BadStubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                let json = [String: AnyObject]()

                return SignalProducer { observer, disposable in
                    sendNext(observer, json)
                    sendCompleted(observer)
                }.observeOn(QueueScheduler())
            }
        }

        class ErrorStubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                return SignalProducer { observer, disposable in
                    sendError(observer, .NotConnectedToInternet)
                }.observeOn(QueueScheduler())
            }
        }

        // MARK: - Spec
        override func spec() {
            it("returns images if the network works correctly.") {
                var response: ResponseEntity? = nil
                let search = ImageSearch(network: GoodStubNetwork())
                search.searchImages()
                    .on(next: { response = $0 })
                    .start()

                expect(response).toEventuallyNot(beNil())
                expect(response?.totalCount).toEventually(equal(123))
                expect(response?.images.count).toEventually(equal(2))
                expect(response?.images[0].id).toEventually(equal(0))
                expect(response?.images[1].id).toEventually(equal(1))
            }
            it("sends an error if the network returns incorrect data.") {
                var error: NetworkError? = nil
                let search = ImageSearch(network: BadStubNetwork())
                search.searchImages()
                    .on(error: { error = $0 })
                    .start()

                expect(error).toEventually(equal(NetworkError.IncorrectDataReturned))
            }
            it("passes the error sent by the network.") {
                var error: NetworkError? = nil
                let search = ImageSearch(network: ErrorStubNetwork())
                search.searchImages()
                    .on(error: { error = $0 })
                    .start()

                expect(error).toEventually(equal(NetworkError.NotConnectedToInternet))
            }
        }
    }

At the beginning, three stubs are defined. `GoodStubNetwork` returns a `SignalProducer` that sends correct JSON data. `BadStubNetwork` returns a `SignalProducer` that sends incorrect JSON data as an empty dictionary. `ErrorStubNetwork` returns a `SignalProducer` that does not send JSON data but an error. All the `SignalProducer`s send the events in a background thread emulating an asynchronous network response by specifying `.observeOn(QueueScheduler())`.

In `spec()`, three unit tests (or specs) are defined. The first one checks the case that `attemptMap` successfully converts JSON data to a `ResponseEntity`. The second one checks the case that `attemptMap` converts JSON data to an error. The third one checks the case that the error sent from `ErrorStubNetwork` is passed through `ImageSearch`.

Type `Command-U` to run the tests. Passed! We finished implementing the main part of our model in MVVM architecture.

## Conclusion

Through the development of the Model part of the example app in MVVM architecture, we learned how to design the Model to decouple from ViewModel and the external system. Protocols were used to remove the direct dependencies on the implementations. ReactiveCocoa was used to handle events between the decoupled components. Also we found Himotoki was helpful to map JSON data to our entities. In the next blog post, we will design and implement View and ViewModel parts.

If you have questions, suggestions or problems, feel free to leave comments.

[^1]: If you are familiar with [functional programming](https://en.wikipedia.org/wiki/Functional_programming), [Argo](https://github.com/thoughtbot/Argo) is also a good choice for a JSON parser.
[^2]: In [DDD (Domain-Driven Design)](https://en.wikipedia.org/wiki/Domain-driven_design), an entity is a concept of a domain model and is defined by its identity. In our project, the term is used to mean just a concept or object regardless of its identity.
[^3]: Refer to [this page](http://programmers.stackexchange.com/questions/151733/if-immutable-objects-are-good-why-do-people-keep-creating-mutable-objects) to know more about immutability and mutability.
[^4]: "[How to Implement the ErrorType Protocol](https://realm.io/news/testing-swift-error-type/)" is worth reading about `ErrorType`.
[^5]: `try?` is used just to simplify this blog post. `do-try-catch` should be used to handle an error to get informative.
