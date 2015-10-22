+++
date = "2015-10-22T19:16:58+09:00"
slug = "summary-of-the-22nd-potatotips-meetup-in-tokyo"
tags = ["potatotips", "meetup", "tokyo", "ios", "android"]
title = "Summary of the 22nd potatotips meetup in Tokyo"

+++

I attended [the 22nd potatotips meetup for iOS/Android developers in Tokyo](https://translate.google.com/translate?sl=ja&tl=en&js=y&prev=_t&hl=ja&ie=UTF-8&u=http%3A%2F%2Fconnpass.com%2Fevent%2F20240%2F&edit-text=&act=url) on October 13th, 2015. It was held in an office of [Mercari](https://www.mercari.com). Tips in variety of fields of iOS/Android development were shared in the meetup. The talks were in the local language, but the tips were worth sharing here. I summarized the tips in this blog[^1].

I was personally interested in the following talks.

- [My Personal App Chosen as One of Best New Apps](#my-personal-app)
- [Comparison of RxSwift, ReactKit and ReactiveCocoa on Swift 2](#reactive-frameworks)
- [Easy and Practical Deep Learning App with Caffe](#caffe)


## iOS

### IBDesignable x PaintCode
*Presented by [86](https://github.com/86) with [a slide in the local language](https://speakerdeck.com/86/ibdesignable-x-paintcode)*

The presenter introduced [PaintCode](http://www.paintcodeapp.com), a vector drawing tool that can export design code of UI to Xcode. The exported code works with `IBDesignable` to update the rendering of graphics in Interface Builder. The presenter said the scheme is still not used in products of [Mercari](https://www.mercari.com), where he works at, but trying and investigating that kind of tools gives a better collaboration way between designers and engineers.

The sample Xcode project is available at [a repository in GitHub](https://github.com/86/starkit).

### Introducing Cardio
*Presented by [kitasuke](https://github.com/kitasuke)*

<iframe src="//www.slideshare.net/slideshow/embed_code/key/JzGiY3DIh6S1y5" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kitasuke/introducing-cardio" title="Introducing Cardio" target="_blank">Introducing Cardio</a> </strong> from <strong><a href="//www.slideshare.net/kitasuke" target="_blank">Yusuke Kita</a></strong> </div>

The presenter also works at Mercari. He developed a HealthKit wrapper framework named [Cardio](https://github.com/kitasuke/Cardio) as his personal project. The problem of HealthKit is that it has too many classes to handle. He wrote Cardio to simplify the HealthKit usage specialized to workout apps.

Cardio has an API that takes advantage of protocol extension in Swift 2. You extend `Context` protocol of Cardio to configure HealthKit features as shown in 16th page of the slide above. Reading the code gave me good insight to design APIs in Swift 2.

### My First tvOS
*Presented by [TachibanaKaoru](https://github.com/toyship) with [a slide in the local language](http://www.slideshare.net/toyship/my-first-tvos)*

Apple TV and its development were introduced. Only a few developers in the meetup had/saw Apple TV, and the introduction by the talker who actually has one was interesting for the audience.

What you have to care to develop an tvOS app are:

- So called 10 feet UI
- Difficult text input
- Limited resource
- Shared device within a family or members unlike iPhone, a personal device

### <a name="my-personal-app"></a>My Personal App Chosen as One of Best New Apps
*Presented by [mnat44](https://github.com/mnat44) with [a slide in the local language](http://www.slideshare.net/motokinarita7/ss-53871895)*

The presenter talked his experience that his personally-developed app was chosen as one of Best New Apps in the App Store. He developed his idea and prototype of the app, [Revolver Camera](https://itunes.apple.com/app/ribokame-revolver-camera/id1039880433?mt=8) since February 2013. Then he paused the development and restarted in the middle of August, 2015 to release with watchOS 2 feature. He is smart taking the strategy to adopt watchOS 2.

He talked his workflow to motivate himself to finish developing it within a month.

- Ticket management with GitHub and Evernote
- Paper prototyping, Illustrator design and implementation
- Limit what the app should do

### Objective-C Generics
*Presented by [gooichi](https://github.com/gooichi) with [a slide in the local language](http://www.slideshare.net/GoichiHirakawa/objectivec-generics)*

The presenter, who has deep knowledge about Objective-C since NextStep, talked about Generics in Objective-C. Xcode 7 added the following new features to Objective-C.

- **Generics**
- Nullability
- KindOf Types
- New macros

Objective-C generics has the following characteristics, and it is worth adding generics to your framework written in Objective-C if it is too big to rewrite in Swift.

- Lightweight implementation of generics compared with the other languages
- Strongly-typed collection
- Importable to Swift
- Static type checking (compiler warnings)
- Backward compatibility

### <a name="reactive-frameworks"></a>Comparison of RxSwift, ReactKit and ReactiveCocoa on Swift 2
*Presented by [ooba](https://github.com/bricklife) with [a slide in the local language](http://yoneapp.hatenablog.com/entry/2015/10/14/014258)*

The three major reactive programming frameworks in Swift are [RxSwift](https://github.com/ReactiveX/RxSwift), [ReactKit](https://github.com/ReactKit/ReactKit) and [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa). The presenter compared the three to pick up one to use in a project.

Sample code

- RxSwift: Official sample code is in the repo
- ReactKit: Official sample code is in ReactKitCatalog repo
- ReactiveCocoa: No official sample code

Databinding (replacing KVO)

- RxSwift: `Variable` type
- ReactKit: Use NSObject KVO with `dynamic` keyword
- ReactiveCocoa: `MutableProperty`

I chose ReactiveCocoa to write [an example project](https://github.com/Swinject/SwinjectMVVMExample) of [Swinject](https://github.com/Swinject/Swinject), but the others also have their own advantages to use in your projects.

### Tips for iOS Hackathon Beginners
*Presented by [satoshi0212](https://github.com/satoshi0212)*

The presenter talked about tips to join a hackathon for beginners. It was fun to listen to his talk.

Required skills as an iOS engineer:

- API call to a server (e.g. [Alamofire](https://github.com/Alamofire/Alamofire) or [APIKit](https://github.com/ishkawa/APIKit))
- JSON response handling (e.g. [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) or [Himotoki](https://github.com/ikesyo/Himotoki))
- UI implementation to display the response

Advances skills:

- Machine learning
- Device hacking
- Unity

Before a hackathon:
- Prepare your idea
- Check the idea or its implementation do not exist already

At a hackathon:

- Be confident (iOS engineer is popular)
- Keep your output simple
- Get advice from reviewers and mention about it at a presentation (**like hackathonhack**)

What you achieve through a hackathon

- Connection with people in other fields (like musician)
- Connection with corporate recruiters

### <a name="caffe"></a>Easy and Practical Deep Learning App with Caffe
*Presented by [noradaiko](https://github.com/noradaiko) with [a slide in the local language](http://www.slideshare.net/takuyamatsuyama/caffe-potatotips)*

The presenter talked about deep learning using [Caffe framework](http://caffe.berkeleyvision.org) with [his sample project](https://github.com/noradaiko/caffe-ios-sample).

Some features of Caffe introduced in the presentations are:

- GPU calculation
- Only 1-2 sec to run single recognition on iPhone5s with his example
- Working locally (without a server)

It was interesting to see that deep learning calculation works on a mobile device within a reasonable period on a mobile device. It looks expanding idea to develop an app handling images.

### Type Safe Resource Handling in Swift
*Presented by [tasanobu](https://github.com/tasanobu) with [a slide in the local language](http://www.slideshare.net/tasanobu/type-safe-assets-handling-in-swift)*

The presenter introduced [SwiftGen](https://github.com/AliSoftware/SwiftGen) to avoid using strings to specify resources like assets or storyboards.

The problem with those strings is that a typo in the strings cannot be detected by compiler but just causes a failure in run time. SwiftGen generates code to retrieve resources with enums in a type-safe manner.

SwiftGen currently supports:

- Asset catalog
- UIStoryboard
- Localizable.strings
- UIColor

### TinderUI and Scroll
*Presented by [takahito_morinaga_5](https://github.com/ipacho) with [a slide in the local language](http://www.slideshare.net/takahitomorinaga5/tinderui)*

The presenter shared his experience to implement [Tinder](https://www.gotinder.com) swipe UI with vertical scroll. He thought the implementation was not so difficult at the beginning, but actually struggled with some implementation barriers with limitations of touch events.

The slide contains tips or workarounds to the collisions of UIPanGestureRecognizer and UIScrollView events. If you have the same problem, try checking [the slide](http://www.slideshare.net/takahitomorinaga5/tinderui) with [Google Translate](https://translate.google.com).

## Android

Let me excuse to only simply summarize the talks about Android. Some presentations have English slides, so please refer to them if you are interested.

### Tips for better CI on Android
*Presented by [tomoima525](https://github.com/tomoima525)*

<iframe src="//www.slideshare.net/slideshow/embed_code/key/pGq00XEVHGIPHf" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/tomoakiimai2/tips-for-better-ci-on-android" title="Tips for better CI on Android" target="_blank">Tips for better CI on Android</a> </strong> from <strong><a href="//www.slideshare.net/tomoakiimai2" target="_blank">Tomoaki Imai</a></strong> </div>

The presentation contains tips to use both Travis CI and Circle CI to split tasks into them to run in parallel.

### Groovy/Spock for Android app testing
*Presented by [izumin5210](https://github.com/izumin5210)*

<script async class="speakerdeck-embed" data-id="48b9734572ed4c5ab69d1754bcb83f92" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

RSpec-like testing framework [Spock](http://spockframework.org) was introduced. It has parameterized test with simple descriptions and messages like [power-assert](https://github.com/power-assert-js/power-assert).

### Regexp in Android and Java
*Presented by [KeithYokoma](https://github.com/KeithYokoma)*

<script async class="speakerdeck-embed" data-id="6791ff09049b415dbe14e05751312f00" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

The presenter talked about differences between Android and Java, especially in regular expressions. The regular expression engines implemented in them are different, causing different behavior.

### A Primer of Scketch3 for Engineers (2nd)
*Presented by [androhi](https://github.com/androhi) with [a slide in the local language](https://speakerdeck.com/androhi/sok-enziniafalsetamefalsesketch3ru-men)*

[Sketch3](http://www.sketchapp.com) was introduced in the talk. The presenter told the tool is good not only for designers but engineers. What I was interested in was `sketchtool` command, which might be able to combine with [fastlane](https://github.com/KrauseFx/fastlane).

### Hand-made Crash Report of Android Library
*Presented by [petitviolet](https://github.com/petitviolet) with [a slide in the local language](https://speakerdeck.com/petitviolet/hand-made-crash-report-of-android-library)*

The presenter shared his experience to develop his own crash reporter to report crashes only within libraries embedded in an app.

### Investigation of Uploading/Downloading
*Presented by [rejasupotaro](https://github.com/rejasupotaro)*

The presenter shared his investigation on data upload/download especially for targeting low bandwidth countries. Image format using WEBP instead of JPEG was interesting.

### Encryption Library Conceal
*Presented by [ksk_kbys](https://github.com/kobakei) with [a slide in the local language](https://speakerdeck.com/kobakei/an-hao-hua-raiburariconceal)*

The presenter introduced [Conceal](https://github.com/facebook/conceal) to store sensitive data in a mobile app.

[^1]: Some slides are written in the local language. Try reading them with [Google Translate](https://translate.google.com) if you are interested.
