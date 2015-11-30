+++
date = "2015-09-10T11:02:53+09:00"
slug = "dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-5-asynchronous-image-load"
tags = ["swinject", "dependency-injection", "mvvm", "reactivecocoa", "swift", "alamofire"]
title = "Dependency Injection in MVVM Architecture with ReactiveCocoa Part 5: Asynchronous Image Load"

+++

- **Updated on Nov 20, 2015** to migrate to ReactiveCocoa v4.0.0 alpha 3 and Alamofire v3.x.
- **Updated on Oct 1, 2015** for the release versions of Swift 2 and Xcode 7.

By [the previous blog post](/post/dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-4-implementing-the-view-and-viewmodel/), we developed an example app, in MVVM architecture, displaying meta data of images received from [Pixabay](https://pixabay.com/) server. In this blog post, we are going add a feature to asynchronously load the images. To handle the asynchronous events, [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) will be used as we did in the previous posts. Through the development, we will learn how to add a feature in MVVM architecture with unit tests and dependency injection are updated.

The source code used in the blog posts is available at:

- [SwinjectMVVMExample](https://github.com/Swinject/SwinjectMVVMExample): Complete version of the project.
- [SwinjectMVVMExample_ForBlog](https://github.com/yoichitgy/SwinjectMVVMExample_ForBlog): Simplified version of the project to follow the blog posts (except updates of Xcode and frameworks).

## Notice

To use with Swift 2 and Xcode 7, ReactiveCocoa [version 4.0](https://github.com/ReactiveCocoa/ReactiveCocoa/releases) is used though it is still in an alpha version at the moment. Notice that ReactiveCocoa 3.0 has functional style APIs like `|> map` or `|> flatMap`, but version 4 APIs are in protocol oriented and fluent style like `.map()` or `.flatMap()`.

## Model

First, we are going to add the feature to request an image to our Model. Add `requestImage` method to `Networking` protocol. The method takes an image URL and returns a SignalProducer instance to send the image.

**Networking.swift**

    import ReactiveCocoa

    public protocol Networking {
        // Omitted

        func requestImage(url: String) -> SignalProducer<UIImage, NetworkError>
    }

Modify `Network` class to implement `requestImage` method. In the method, the initializer of `SignalProducer` with a trailing closure is used to convert an asynchronous response of Alamofire to a `Signal` of ReactiveCocoa. If the response is successful and its data are valid, `.Next` and `.Completed` events are sent to `observer`. Otherwise an `.Failed` event is sent. Because the response of Alamofire runs in the main thread by default, a serial queue is passed to Alamofire.

**Network.swift**

```
import ReactiveCocoa
import Alamofire

public final class Network: Networking {
    private let queue = dispatch_queue_create(
        "SwinjectMMVMExample.ExampleModel.Network.Queue",
        DISPATCH_QUEUE_SERIAL)

    // Omitted

    public func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
        return SignalProducer { observer, disposable in
            let serializer = Alamofire.Request.dataResponseSerializer()
            Alamofire.request(.GET, url)
                .response(queue: self.queue, responseSerializer: serializer) {
                    response in
                    switch response.result {
                    case .Success(let data):
                        guard let image = UIImage(data: data) else {
                            observer.sendFailed(.IncorrectDataReturned)
                            return
                        }
                        observer.sendNext(image)
                        observer.sendCompleted()
                    case .Failure(let error):
                        observer.sendFailed(NetworkError(error: error))
                    }
            }
        }
    }
}
```

Update stubs used in `ImageSearchSpec` since `requestImage` method was added to `Networking` protocol. Because `requestImage` method is not used in the unit tests, each stub method just returns an empty `SignalProducer`.

**ImageSearchSpec.swift**

    import Quick
    import Nimble
    import ReactiveCocoa
    @testable import ExampleModel

    class ImageSearchSpec: QuickSpec {
        // MARK: Stub
        class GoodStubNetwork: Networking {
            // Omitted

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer.empty
            }
        }

        class BadStubNetwork: Networking {
            // Omitted

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer.empty
            }
        }

        class ErrorStubNetwork: Networking {
            // Omitted

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer.empty
            }
        }

        // Omitted
    }

Add unit tests to `NetworkSpec` to check the new `requestImage` method. As a stable server for the tests, [httpbin.org](http://httpbin.org) is used.

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

            // Omitted

            describe("Image") {
                it("eventually gets an image.") {
                    var image: UIImage?
                    network.requestImage("https://httpbin.org/image/jpeg")
                        .on(next: { image = $0 })
                        .start()

                    expect(image).toEventuallyNot(beNil(), timeout: 5)
                }
                it("eventually gets an error if incorrect data for an image is returned.") {
                    var error: NetworkError?
                    network.requestImage("https://httpbin.org/get")
                        .on(failed: { error = $0 })
                        .start()

                    expect(error).toEventually(
                        equal(NetworkError.IncorrectDataReturned), timeout: 5)
                }
                it("eventually gets an error if the network has a problem.") {
                    var error: NetworkError? = nil
                    network.requestImage("https://not.existing.server.comm/image/jpeg")
                        .on(failed: { error = $0 })
                        .start()

                    expect(error).toEventually(
                        equal(NetworkError.NotReachedServer), timeout: 5)
                }
            }
        }
    }

The first test checks that `Network` returns an image asynchronously as a successful case. The second test checks that `Network` sends `NetworkError.IncorrectDataReturned` error if non-image data are returned from the server. The third test checks that an error from Alamofire is converted to its corresponding `NetworkError` and passed through the `Network` instance.

Type `Command-U` to run the unit tests.

## ViewModel

Let's move on to our ViewModel to receive the image from Model and to handle it for View. At the beginning, add `RACUtil.swift` with the following content to `ExampleViewModel` group. Make sure that it is added to `ExampleViewModel` target when you save them.

**RACUtil.swift**

    import Foundation
    import ReactiveCocoa

    internal extension NSObject {
        internal var racutil_willDeallocProducer: SignalProducer<(), NoError>  {
            return self.rac_willDeallocSignal()
                .toSignalProducer()
                .map { _ in }
                .flatMapError { _ in SignalProducer(value: ()) }
        }
    }

In the extension to `NSObject`, `rac_willDeallocSignal` is mapped to a `SignalProducer` that sends an event with an empty tuple when an object is being deallocated. We add the extension here because ReactiveCocoa Swift APIs still do not have an extension equivalent to `rac_willDeallocSignal` supported by Objective-C APIs. `toSignalProducer` is used to convert the Objective-C `Signal` to Swift `SignalProducer`, and `map` and `flatMapError` are used to transform the event and error types.

Add `getPreviewImage` method to `ImageSearchTableViewCellModeling` protocol and `ImageSearchTableViewCellModel` class as followings.

**ImageSearchTableViewCellModeling.swift**

    import ReactiveCocoa

    public protocol ImageSearchTableViewCellModeling {
        var id: UInt64 { get }
        var pageImageSizeText: String { get }
        var tagText: String { get }

        func getPreviewImage() -> SignalProducer<UIImage?, NoError>
    }

**ImageSearchTableViewCellModel.swift**

    import ReactiveCocoa
    import ExampleModel

    public final class ImageSearchTableViewCellModel
        : NSObject, ImageSearchTableViewCellModeling
    {
        public let id: UInt64
        public let pageImageSizeText: String
        public let tagText: String

        private let network: Networking
        private let previewURL: String
        private var previewImage: UIImage?

        internal init(image: ImageEntity, network: Networking) {
            id = image.id
            pageImageSizeText = "\(image.pageImageWidth) x \(image.pageImageHeight)"
            tagText = image.tags.joinWithSeparator(", ")

            self.network = network
            previewURL = image.previewURL

            super.init()
        }

        public func getPreviewImage() -> SignalProducer<UIImage?, NoError> {
            if let previewImage = self.previewImage {
                return SignalProducer(value: previewImage).observeOn(UIScheduler())
            }
            else {
                let imageProducer = network.requestImage(previewURL)
                    .takeUntil(self.racutil_willDeallocProducer)
                    .on(next: { self.previewImage = $0 })
                    .map { $0 as UIImage? }
                    .flatMapError { _ in SignalProducer<UIImage?, NoError>(value: nil) }

                return SignalProducer(value: nil)
                    .concat(imageProducer)
                    .observeOn(UIScheduler())
            }
        }
    }

`getPreviewImage` method returns a `SignalProducer` instance sending `UIImage`. A cached image, as `previewImage` property, is wrapped in an `SignalProducer` instance if the cache exists. Otherwise, another `SignalProducer` that requests an image to `Networking` is returned.

The latter `SignalProducer` consists of two parts concatenated by `concat` method. The first part is `SignalProducer(value: nil)`, which sends `nil` then completes immediately. The `nil` is sent at the beginning to clear an old image displayed in the `UIImageView` on a reused cell. The second part is `imageProducer`, which requests the image to `Networking`. In the second part, an error is mapped to `nil` by `flatMapError` to ignore the error because no error message should be displayed for each cell. To terminate the signal producer when the `ImageSearchTableViewCellModel` instance is deallocated, `takeUntil` method is called with `racutil_willDeallocProducer`. To use the `NSObject` extension, `ImageSearchTableViewCellModel` inherits `NSObject`.

Modify `ImageSearchTableViewModel` to pass a `Networking` instance as below. A parameter is added to its initializer to get `Networking` instance injected. The instance is passed to the initializer of `ImageSearchTableViewCellModel` in `startSearch` method.

**ImageSearchTableViewModel.swift**

```
import ReactiveCocoa
import ExampleModel

public final class ImageSearchTableViewModel: ImageSearchTableViewModeling {
    public var cellModels: AnyProperty<[ImageSearchTableViewCellModeling]> {
        return AnyProperty(_cellModels)
    }
    private let _cellModels = MutableProperty<[ImageSearchTableViewCellModeling]>([])
    private let imageSearch: ImageSearching
    private let network: Networking

    public init(imageSearch: ImageSearching, network: Networking) {
        self.imageSearch = imageSearch
        self.network = network
    }

    public func startSearch() {
        imageSearch.searchImages()
            .map { response in
                response.images.map {
                    ImageSearchTableViewCellModel(image: $0, network: self.network)
                        as ImageSearchTableViewCellModeling
                }
            }
            .observeOn(UIScheduler())
            .on(next: { cellModels in
                self._cellModels.value = cellModels
            })
            .start()
    }
}
```

At last, modify `AppDelegate` to add dependency injection of `Networking` to `ImageSearchTableViewModel` as shown below.

**AppDelegate.swift**

    @UIApplicationMain
    class AppDelegate: UIResponder, UIApplicationDelegate {
        var window: UIWindow?
        let container = Container() { container in
            // Models
            container.register(Networking.self) { _ in Network() }
            container.register(ImageSearching.self) { r in
                ImageSearch(network: r.resolve(Networking.self)!)
            }

            // View models
            container.register(ImageSearchTableViewModeling.self) { r in
                ImageSearchTableViewModel(
                    imageSearch: r.resolve(ImageSearching.self)!,
                    network: r.resolve(Networking.self)!)
            }

            // Views
            container.registerForStoryboard(ImageSearchTableViewController.self) { r, c in
                c.viewModel = r.resolve(ImageSearchTableViewModeling.self)!
            }
        }

        // Omitted
    }

Let's modify and add unit tests for the updated ViewModel. First, modify `ImageSearchTableViewModelSpec` to add `StubNetwork` and pass its instance to the modified initializer of `ImageSearchTableViewModel`.

**ImageSearchTableViewModelSpec.swift**

    class ImageSearchTableViewModelSpec: QuickSpec {
        // MARK: Stub
        class StubImageSearch: ImageSearching {
            func searchImages() -> SignalProducer<ResponseEntity, NetworkError> {
                return SignalProducer { observer, disposable in
                    observer.sendNext(dummyResponse)
                    observer.sendCompleted()
                }
                .observeOn(QueueScheduler())
            }
        }

        class StubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                return SignalProducer.empty
            }

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer.empty
            }
        }

        // MARK: Spec
        override func spec() {
            var viewModel: ImageSearchTableViewModel!
            beforeEach {
                viewModel = ImageSearchTableViewModel(
                    imageSearch: StubImageSearch(),
                    network: StubNetwork())
            }

            // Omitted
        }
    }

Add a dummy image instance to `DummyResponse.swift`. The instance will be used later.

**DummyResponse.swift**

    let image1x1: UIImage = {
        UIGraphicsBeginImageContextWithOptions(CGSizeMake(1, 1), true, 0)
        let image = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        return image
    }()

Add stubs and unit tests to `ImageSearchTableViewCellModelSpec` as below.

**ImageSearchTableViewCellModelSpec.swift**

    import Foundation
    import Quick
    import Nimble
    import ReactiveCocoa
    @testable import ExampleModel
    @testable import ExampleViewModel

    class ImageSearchTableViewCellModelSpec: QuickSpec {
        // MARK: Stubs
        class StubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                return SignalProducer.empty
            }

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer(value: image1x1).observeOn(QueueScheduler())
            }
        }

        class ErrorStubNetwork: Networking {
            func requestJSON(url: String, parameters: [String : AnyObject]?)
                -> SignalProducer<AnyObject, NetworkError>
            {
                return SignalProducer.empty
            }

            func requestImage(url: String) -> SignalProducer<UIImage, NetworkError> {
                return SignalProducer(error: .NotConnectedToInternet)
            }
        }

        // MARK: Spec
        override func spec() {
            var viewModel: ImageSearchTableViewCellModel!
            beforeEach {
                viewModel = ImageSearchTableViewCellModel(
                    image: dummyResponse.images[0],
                    network: StubNetwork())
            }

            describe("Constant values") {
                it("sets id.") {
                    expect(viewModel.id).toEventually(equal(10000))
                }
                it("formats tag and page image size texts.") {
                    expect(viewModel.pageImageSizeText)
                        .toEventually(equal("1000 x 2000"))
                    expect(viewModel.tagText).toEventually(equal("a, b"))
                }
            }
            describe("Preview image") {
                it("returns nil at the first time.") {
                    var image: UIImage? = image1x1
                    viewModel.getPreviewImage()
                        .take(1)
                        .on(next: { image = $0 })
                        .start()

                    expect(image).toEventually(beNil())
                }
                it("eventually returns an image.") {
                    var image: UIImage? = nil
                    viewModel.getPreviewImage()
                        .on(next: { image = $0 })
                        .start()

                    expect(image).toEventuallyNot(beNil())
                }
                it("returns an image on the main thread.") {
                    var onMainThread = false
                    viewModel.getPreviewImage()
                        .skip(1) // Skips the first nil.
                        .on(next: { _ in onMainThread = NSThread.isMainThread() })
                        .start()

                    expect(onMainThread).toEventually(beTrue())
                }
                context("with an image already downloaded") {
                    it("immediately returns the image omitting the first nil.") {
                        var image: UIImage? = nil
                        viewModel.getPreviewImage().start(completed: {
                            viewModel.getPreviewImage()
                                .take(1)
                                .on(next: { image = $0 })
                                .start()
                        })

                        expect(image).toEventuallyNot(beNil())
                    }
                }
                context("on error") {
                    it("returns nil.") {
                        var image: UIImage? = image1x1
                        let viewModel = ImageSearchTableViewCellModel(
                            image: dummyResponse.images[0],
                            network: ErrorStubNetwork())
                        viewModel.getPreviewImage()
                            .skip(1) // Skips the first nil.
                            .on(next: { image = $0 })
                            .start()

                        expect(image).toEventually(beNil())
                    }
                }
            }
        }
    }

`requestImage` method of `StubNetwork` returns a `SignalProducer` sending the dummy image. That of `ErrorStubNetwork` returns a `SignalProducer` sending an error. Before adding new unit tests, refactoring is done on `spec`. The existing tests are grouped within `describe("Constant values")`.

Five new unit tests for `getPreviewImage` method are added to `describe("Preview image")` group. The first test checks the `SignalProducer` sends `nil` as its first event. The second test checks it sends an image as the succeeding event. The third test checks the image event is sent on the main thread. The forth test checks a cached image is sent immediately in case the cache exists. The test is grouped within `context` because the test is in the certain condition. The fifth test checks that an error on `Networking` instance is converted and sent as `nil`. This test is also grouped within `context`.

Input `Command-U` and run the unit tests. Move on to the next section to implement our View.

## View

First, we are going to add an extension to `UITableViewCell` in the same way as we did to `NSObject`. Add `RACUtil.swift` with the following content to `ExampleView` group. Make sure that it is added to `ExampleView` target. In the extension, `rac_prepareForReuseSignal`, as ReactiveCocoa Objective-C API, is transformed to a corresponding Swift instance. It sends an event with an empty tuple when `prepareForReuse` of `UITableViewCell` is invoked.

**RACUtil.swift**

    import UIKit
    import ReactiveCocoa

    internal extension UITableViewCell {
        internal var racutil_prepareForReuseProducer: SignalProducer<(), NoError>  {
            return self.rac_prepareForReuseSignal
                .toSignalProducer()
                .map { _ in }
                .flatMapError { _ in SignalProducer(value: ()) }
        }
    }

Second, modify `ImageSearchTableViewCell` to update the image view when `viewModel` property is set. The signal of `getPreviewImage` is terminated upon reuse of the cell for another table row to avoid updating the cell with the image used for the previous row.

**ImageSearchTableViewCell.swift**

    import UIKit
    import ExampleViewModel
    import ReactiveCocoa

    internal final class ImageSearchTableViewCell: UITableViewCell {
        internal var viewModel: ImageSearchTableViewCellModeling? {
            didSet {
                tagLabel.text = viewModel?.tagText
                imageSizeLabel.text = viewModel?.pageImageSizeText

                if let viewModel = viewModel {
                    viewModel.getPreviewImage()
                        .takeUntil(self.racutil_prepareForReuseProducer)
                        .on(next: { self.previewImageView.image = $0 })
                        .start()
                }
                else {
                    previewImageView.image = nil
                }
            }
        }

        @IBOutlet weak var previewImageView: UIImageView!
        @IBOutlet weak var tagLabel: UILabel!
        @IBOutlet weak var imageSizeLabel: UILabel!
    }

Type `Command-R` and run the app. You will see each image view is filled like the following image.

![SwinjectMVVMExample Images Displayed in Table View Cells](/images/post/2015-09/SwinjectMVVMExampleCellsWithImagesScreenshot.png)

At last, add `ImageSearchTableViewCellSpec.swift` with the following content to `ExampleViewTests` group. Make sure that it is added to `ExampleViewTests` target.

**ImageSearchTableViewCellSpec.swift**

    import Quick
    import Nimble
    import ReactiveCocoa
    import ExampleViewModel
    @testable import ExampleView

    class ImageSearchTableViewCellSpec: QuickSpec {
        class MockViewModel: ImageSearchTableViewCellModeling {
            let id: UInt64 = 0
            let pageImageSizeText = ""
            let tagText = ""

            var getPreviewImageStarted = false

            func getPreviewImage() -> SignalProducer<UIImage?, NoError> {
                return SignalProducer<UIImage?, NoError> { observer, _ in
                    self.getPreviewImageStarted = true
                    observer.sendCompleted()
                }
            }
        }

        override func spec() {
            it("starts getPreviewImage signal producer when its view model is set.") {
                let viewModel = MockViewModel()
                let view = createTableViewCell()

                expect(viewModel.getPreviewImageStarted) == false
                view.viewModel = viewModel
                expect(viewModel.getPreviewImageStarted) == true
            }
        }
    }

    private func createTableViewCell() -> ImageSearchTableViewCell {
        let bundle = NSBundle(forClass: ImageSearchTableViewCell.self)
        let storyboard = UIStoryboard(name: "Main", bundle: bundle)
        let tableViewController = storyboard
            .instantiateViewControllerWithIdentifier("ImageSearchTableViewController")
            as! ImageSearchTableViewController
        return tableViewController.tableView
            .dequeueReusableCellWithIdentifier("ImageSearchTableViewCell")
            as! ImageSearchTableViewCell
    }

The test checks, with a mock of `ImageSearchTableViewCellModeling`, `getPreviewImage` is called when `viewModel` property of `ImageSearchTableViewCell` is set.

Input `Command-U` to run the test. Passed! We have finished implementing the table view displaying images that are asynchronously loaded from the network. Remember now we have not only the implementation but also the unit tests that give us confidence to keep developing working software!

## Conclusion

In this blog post, we implemented the feature to asynchronously load an image to `UIImageView` in MVVM architecture. We learned how to add new methods to protocols and their conforming classes with the update of unit tests in Model, ViewModel and View. The dependency injection to ViewModel was also updated during the implementation. ReactiveCocoa was used throughout the Model, ViewModel and View to pass and handle events in the abstracted way.

Through the series of the blog posts, we learned:

- Part 1: The concepts and basics of MVVM and ReactiveCocoa.
- Part 2: The setup of Xcode project composed of MVVM framework targets with external frameworks installed via [Carthage](https://github.com/Carthage/Carthage).
- Part 3: The model design using protocols to decouple our app from external system, e.g. network.
- Part 4: ViewModel and View implementation with dependencies injected by AppDelegate.
- Part 5: Modification of Model, ViewModel and View to add a new feature with unit tests updated.

We did not only develop the example app but wrote unit tests[^1] that use stubs and mocks of protocols representing MVVM interfaces. Keeping the cycle to add protocols, implementations, and unit tests in MVVM architecture, we are always confident to develop the project further. Decoupling of Model, View and ViewModel by using the abstracted events of [ReactiveCocoa](https://github.com/Swinject/Swinject) and dependency injection with [Swinject](https://github.com/Swinject/Swinject) is the key to the cycle.

The series of the blog posts ends here, but [the project in the GitHub repository](https://github.com/Swinject/SwinjectMVVMExample) has further development to show image details, to receive more image data when scrolled to the bottom of the table, to handle errors and to add localization. If you are interested, check the project. Adding a star to [Swinject project](https://github.com/Swinject/Swinject) is highly appreciated.

[^1]: The blog posts always wrote unit tests after implementing features, but actually tests should be written first or together with the feature implementation.
