+++
date = "2015-08-29T10:00:28+09:00"
slug = "dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-2-project-setup"
tags = ["swinject", "dependency-injection", "mvvm", "reactivecocoa", "swift", "carthage"]
title = "Dependency Injection in MVVM Architecture with ReactiveCocoa Part 2: Project Setup"

+++

In [the last blog post](/post/dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-1-introduction/), the basic concepts of MVVM (Model-View-ViewModel) and [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) were introduced. From this blog post, we are going to develop an example app to demonstrate dependency injection with [Swinject](https://github.com/Swinject/Swinject) in MVVM architecture. We will use ReactiveCococa to handle events passed between MVVM components. In this blog post, you will learn how to setup an Xcode project composing Molel, View and ViewModel as  frameworks.

The example app asynchronously searches, downloads and displays images obtained from [Pixabay](https://pixabay.com) via [its API](https://pixabay.com/api/docs/), as shown in the GIF animation below. The source code used in the blog posts is available at [a repository on GitHub](https://github.com/Swinject/SwinjectMVVMExample).

![SwinjectMVVMExample ScreenRecord](/images/post/2015-08/SwinjectMVVMExampleScreenRecord.gif)

## Requirements

- Swift 2.0
- Xcode 7 beta 6
- [Carthage](https://github.com/Carthage/Carthage) 0.8 or later
- [Pixabay](https://pixabay.com/api/docs/) API username and key

We are going to use Swift 2.0 with Xcode 7 though they are still in beta versions. To use with Swift 2, a development version of ReactiveCocoa in [swift2 branch](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/swift2) will be used. Notice that ReactiveCocoa 3.0 has functional style APIs like `|> map` or `|> flatMap`, but APIs in swift2 branch are in protocol oriented and fluent style like `.map()` or `.flatMap()`. Since swift2 branch is still in development, the APIs might be changed in the future.

Carthage can be installed by [its installer (Carthage.pkg)](https://github.com/Carthage/Carthage/releases), or [Homebrew](http://brew.sh/) with the following commands.

    brew update
    brew install carthage

You can get a free API username and key at [Pixabay](https://pixabay.com/). First, sign up and log in there. Then, access its [API documentation page](https://pixabay.com/api/docs/). Your API username and key will be displayed in "Request parameters" section.

## Project Setup Overview

In [the last blog post](/post/dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-1-introduction/), we learned View depended on ViewModel and ViewModel on Model in MVVM architecture. How can we force the direction of dependencies when we write protocols, classes or structs representing Models, Views or ViewModels? It is easy to get chaotic if all the types are placed in a directory, or even in an application target.

Java or .NET applications are often composed of several [JAR](https://en.wikipedia.org/wiki/JAR_%28file_format%29)s or [DLL](https://en.wikipedia.org/wiki/Dynamic-link_library)s to make the responsibilities of the components explicit. Since iOS 8 has introduced dynamic frameworks, it is easy to setup an iOS app composed of several dynamic frameworks. The architecture of an iOS app containing Model, View and ViewModel frameworks, as diagrammed in the image below, ensures that the direction of the dependencies is from View to ViewModel and ViewModel to Model with the dependencies injected by the application. For example, if you make a type in ViewModel framework, it can reference types in Model framework but cannot those in View framework.

The architecture keeps the consistency of the direction of the dependencies, and makes the app easy to develop, test and maintain.

![App Structure Diagram](/images/post/2015-08/Diagram-MVVMAppStructure.png)

## Project Setup Composing MVVM Frameworks

Let's start creating our Xcode project composing the MVVM frameworks. Select `File > New > Project...` menu and `iOS > Application > Single View Application` item. Set  its product name to `SwinjectMVVMExample`, add `.SwinjectMVVMExample` to the end of the bundle identifier, set language to Swift and devices to iPhone, and check `Include Unit Tests` only[^1]. Save it anywhere in your local storage.

![Screenshot New Project](/images/post/2015-08/SwinjectMVVMExampleScreenshotNewProject.png)

Next we are going to add Model, View and ViewModel frameworks. They will be named `ExampleModel`, `ExampleView` and `ExampleViewModel`, respectively, in our example app. When you create your own app, it is recommended to name the frameworks as `YourAppName` + `Model`, `View`, and `ViewModel`. For exampe, if your app name is `Foobook`, your framework names may be `FoobookModel`, `FoobookView` and `FoobookViewModel`.

Select `File > New > Target...` menu and `iOS > Framework & Library > Cocoa Touch Framework`. Click `Next` button. In the next page, set product name to `ExampleModel`, language to Swift. Keep `Include Unit Test` checked and click `Finish`. In the same way, add `ExampleViewModel` and `ExampleView` framework targets.

![Screenshot New Target](/images/post/2015-08/SwinjectMVVMExampleScreenshotNewTarget.png)

Right click on `SwinjectMVVMExample` in Project Navigator (the left pane in Xcode), and select `New Group`. Set the group name to `Tests`. Drag `ExampleModel`, `ExampleViewModel`, `ExampleView`, `ExampleModelTests`, `ExampleViewModelTests`, `ExampleViewTests` and `SwinjectMVVMExampleTests` in Project Navigator, and place them as shown in the image below. Select `SwinjectMVVMExample` in Project Navigator, and order the targets by dragging as shown in the image.

![Screenshot Project Hierarchy](/images/post/2015-08/SwinjectMVVMExampleScreenshotProjectHierarchy.png)

We are going to configure dependency and link settings of the targets. Select `SwinjectMVVMExample` in Project Navigator, and select `ExampleViewModel` in Targets area. In Build Phases tab, click `+` button in Target Dependencies section, and add `ExampleModel`. Click `+` button in Link Binary with Libraries section, and add `ExampleModel.framework`. In the same way, add `ExampleViewModel` and `ExampleViewModel.framework` to those of `ExampleView` target. Target settings of `ExampleModel` and `SwinjectMVVMExample` can be left because `ExampleModel` has no dependencies and `SwinjectMVVMExample` has those settings by default.

![Screenshot Target Settings ExampleModel](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleModel.png)

![Screenshot Target Settings ExampleViewModel](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleViewModel.png)

![Screenshot Target Settings ExampleView](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleView.png)

![Screenshot Target Settings SwinjectMVVMExample](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsSwinjectMVVMExample.png)

We are going to configure unit test targets too. Select `SwinjectMVVMExample` in Project Navigator, and select `ExampleModelTests` in Targets area. In General tab, set Host Application to `None`. In the same way, set Host Application of `ExampleViewModelTests` and `ExampleViewTests` to `None`.

Again select `ExampleModelTests` in Targets area. In Build Phases tab, remove `SwinjectMVVMExample` in Target Dependencies section by selecting it and click `-` button. In the same remove target dependency on `SwinjectMVVMExample` from `ExampleViewModelTests` and `ExampleViewTests`.

Select `ExampleViewModelTests` in Targets area. In Build Phases tab, add `ExampleModel.framework` to Link Binary with Libraries section. In the same way, add `ExampleViewModel.framework` to that of `ExampleViewTests`. We need those links to stub or mock types in the linked frameworks.

![Screenshot Target Settings ExampleModelTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleModelTests.png)

![Screenshot Target Settings ExampleViewModelTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleViewModelTests.png)

![Screenshot Target Settings ExampleViewTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsExampleViewTests.png)

![Screenshot Target Settings SwinjectMVVMExampleTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotTargetSettingsSwinjectMVVMExampleTests.png)

We are going to setup a build scheme. Click scheme button on the Xcode toolbar, and select `Manage Schemes...` If you see `ExampleModel`, `ExampleViewModel` and `ExampleView` schemes, select them and click `-` button to delete them. Then select `SwinjectMVVMExample` scheme and click `Edit` button. Make sure that `ExampleModelTests`, `ExampleViewModelTests`, `ExampleViewTests` and `SwinjectMVVMExampleTests` are checked in Build and Test settings.

![Screenshot Build Scheme](/images/post/2015-08/SwinjectMVVMExampleScreenshotBuildScheme.png)

![Screenshot Test Scheme](/images/post/2015-08/SwinjectMVVMExampleScreenshotTestScheme.png)

Okay, we have finished setting up the project. Run the app (`Command-R`), then  unit tests (`Command-U`) to check whether the project is configured correctly. If you see an error for an umbrella header, set the target membership of the header file to the target with `Public` accessibility as shown in the image below. Xcode sometimes creates umbrella headers with wrong target memberships.

![Screenshot Umbrella Header](/images/post/2015-08/SwinjectMVVMExampleScreenshotUmbrellaHeader.png)

## Installation of External Frameworks with Carthage

We are going to install [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa), [Himotoki](https://github.com/ikesyo/Himotoki), [Alamofire](https://github.com/Alamofire/Alamofire), [Swinject](https://github.com/Swinject/Swinject), [Quick](https://github.com/Quick/Quick) and [Nimble](https://github.com/Quick/Nimble). We used Alamofire, Quick and Nimble in [the previous example project](/post/dependency-injection-framework-for-swift-simple-weather-app-example-with-swinject-part-1/). This time, we are going to use Himotoki, which is a type-safe JSON decoding library, in place of [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON). The details of Himotoki will be mentioned in the next blog post when we use it.

To install them with [Carthage](https://github.com/Carthage/Carthage), add a text file named `Cartfile` with the following contents. Notice that specific commit versions of some frameworks are specified because they are still in development to support Swift 2 and Xcode 7 (beta).

**Cartfile**

    github "ReactiveCocoa/ReactiveCocoa" "be3d9f5a2ba0fd902c13373e320a4b86f214c1d3"
    github "ikesyo/Himotoki" "a40a9df31ffa2e81daf872e9637170e3708e5478"
    github "Alamofire/Alamofire" ~> 2.0.0
    github "Swinject/Swinject" ~> 0.2.1

    github "Quick/Quick" "1fbcd8a05f6e896e2db66a2e82527b7f24766ef8"
    github "Quick/Nimble" "e3e3978ef610927d70eafd333e162855ad7b6f77"

Run `carthage update --no-build` and `carthage build` in Terminal[^2]. Wait for a couple of minutes (or more) until Carthage finishes building the frameworks[^3]. If you use [Git](https://git-scm.com), [here is a `.gitignore` for Swift](https://github.com/github/gitignore/blob/master/Swift.gitignore) excluding frameworks built by Carthage.

After the build completes, right click on `SwinjectMVVMExample` in Project Navigator, and select `New Group`. Name the group `Frameworks`. Drag and drop the frameworks built in `Carthage/Build/iOS` directory in Finder to the group in Xcode. When you drop them, an action sheet asks you wether they should be added to targets. Uncheck all targets and click `Finish` button.

![Screenshot Umbrella Header](/images/post/2015-08/SwinjectMVVMExampleScreenshotUmbrellaHeaderFrameworksGroup.png)

Select `SwinjectMVVMExample` in Project Navigator, and select `ExampleModel` in Target section with Build Phases tab open. Drag `Alamofire.framework`, `Himotoki.framework`, `ReactiveCocoa.framework` and `Result.framework`[^4] in the `Frameworks` group to Link Binary with Libraries section in Build Phases tab. In the same way, add `ReactiveCocoa.framework` and `Result.framework` to those of `ExampleViewModel` and `ExampleView`. Add `Swinject.framework` to that of `SwinjectMVVMExample`. Add `ReactiveCocoa.framework`, `Result.framework`, `Quick.framework` and `Nimble.framework` to those of `ExampleModelTests`, `ExampleViewModelTests` and `ExampleViewTests`. Add `Swinject.framework`, `Quick.framework` and `Nimble.framework` to that of `SwinjectMVVMExampleTests`. Alamofire and Himotoki are used only in our Model, Swinject only in our application and its test, and Quick and Nimble only in unit tests.

Select `ExampleModelTests` in Target section, click `+` button under Build Phases tab, select `New Copy Files Phase`, and name the new phase `Copy Frameworks`. Within the phase, set Destination to `Frameworks`. Drag `Alamofire.framework`, `Himotoki.framework`, `ReactiveCocoa.framework`, `Result.framework`, `Quick.framework` and `Nimble.framework` in `Frameworks` group on Project Navigator to `Copy Frameworks` phase. In the same way, add `Copy Frameworks` phases to `ExampleViewModelTests` and `ExampleViewTests`, and drag the 6 frameworks to the phases. Add `Copy Frameworks` phase to `SwinjectMVVMExampleTests` in the same way, and add `Quick.framework` and `Nimble.framework` to the phase. Because `SwinjectMVVMExampleTests` is hosted in the app, only Quick and Nimble, which are not used in the app, are added to the phase.

![Screenshot Link Settings ExampleModel](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleModel.png)

![Screenshot Link Settings ExampleViewModel](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleViewModel.png)

![Screenshot Link Settings ExampleView](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleView.png)

![Screenshot Link Settings SwinjectMVVMExample](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsSwinjectMVVMExample.png)

![Screenshot Link Settings ExampleModelTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleModelTests.png)

![Screenshot Link Settings ExampleViewModelTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleViewModelTests.png)

![Screenshot Link Settings ExampleViewTests](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsExampleViewTests.png)

![Screenshot Link Settings SwinjectMVVMExample](/images/post/2015-08/SwinjectMVVMExampleScreenshotLinkSettingsSwinjectMVVMExampleTests.png)

The last settings are as specified in [the README of Carthage](https://github.com/Carthage/Carthage). Select `SwinjectMVVMExample` in Project Navigator, and select `SwinjectMVVMExample` in Targets section with Build Phases tab open. Click `+` button under the tab, and select `New Run Script Phase`. Rename the phase to `Run Script for Frameworks by Carthage` and add the following line to the script text field.

    /usr/local/bin/carthage copy-frameworks

Then add the following paths to Input Files section by clicking `+` button.

    $(SRCROOT)/Carthage/Build/iOS/Alamofire.framework
    $(SRCROOT)/Carthage/Build/iOS/Himotoki.framework
    $(SRCROOT)/Carthage/Build/iOS/ReactiveCocoa.framework
    $(SRCROOT)/Carthage/Build/iOS/Result.framework
    $(SRCROOT)/Carthage/Build/iOS/Swinject.framework

Again click the `+` button under Build Phases tab, select `New Copy Files Phase` and name the phase `Copy dSYMs`. Set its destination to `Products Directory`, and drag `Alamofire.framework.dSYM`, `Himotoki.framework.dSYM`, `ReactiveCocoa.framework.dSYM`, `Result.framework.dSYM` and `Swinject.framework.dSYM` from `Carthage/Build/iOS/` directory in Finder to the `Copy dSYMs` phase in Xcode. Right click on `SwinjectMVVMExample` in Project Navigator and select `New Group`. Name the new group `dSYMs` and put the dSYM files that were added to Project Navigator automatically into the group to tidy up.

![Screenshot dSYMs Group](/images/post/2015-08/SwinjectMVVMExampleScreenshotCarthageScriptAnddSYMs.png)

![Screenshot dSYMs Group](/images/post/2015-08/SwinjectMVVMExampleScreenshotdSYMsGroup.png)

Configurations! You got the project set up. Try running the app and unit tests by `Command-R` and `Command-U` to confirm the project is setup as intended. If you get an error, download [the project on GitHub](https://github.com/Swinject/SwinjectMVVMExample) and compare your project with it to investigate what is wrong.

## Conclusion

We found that the architecture of the app composed of Model, View and ViewModel frameworks ensured the direction of dependencies was consistent from View to ViewModel and ViewModel to Model. We setup the Xcode project in the MVVM architecture, and installed some external frameworks with Carthage. In [the next blog post](/post/dependency-injection-in-mvvm-architecture-with-reactivecocoa-part-3-designing-the-model/), we will start developing the app with ReactiveCocoa and Swinject to take advantage of the MVVM architecture.

If you have questions, suggestions or problems, feel free to leave comments.

[^1]: UI tests are excluded because still Xcode 7 is beta. This blog post will be updated to include them after Xcode 7 is officially released.
[^2]: `carthage update --no-build` and `carthage build` commands are separately used to avoid downloading zipped frameworks built with an older beta version of Xcode. If you install official release versions of the frameworks for an official release version of Xcode, you can just run `carthage update`.
[^3]: If you get an error on `carthage build` command, run it again with `--verbose` as `carthage build --verbose` to investigate the problem.
[^4]: `Result.framework` is used by ReactiveCocoa.
