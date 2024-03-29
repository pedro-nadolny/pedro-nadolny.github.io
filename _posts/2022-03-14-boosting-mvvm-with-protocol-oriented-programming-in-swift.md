---
layout: post
title: Boosting MVVM with Protocol Oriented Programming in Swift
date: 2022-03-14 22:56 -0300
comments: true
---

Recently I was playing around with protocol oriented programming and decided to setup and architecture a sample ios project using that as a main principle. Then I got the idea that I could improve the approach of breaking up view-models in workers/useCases with it, so I've implemented [this sample app](https://github.com/pedro-nadolny/swift-mvvm-protocol-oriented) and this post will attempt to explain the idea within it. 

MVVM based on protocol oriented programming means that instead of having a whole view-model defined in one place, we instead break it up in protocols, each with its implementation of what would be a single exposed method of the view-model, and which I'll be naming then as features from now on. This features will then be composed into the view-models of our application. 

This technique helps maximize the separation of concerns in the business logic layer, and maximize its reusability. Each view-model is a composition of features, and each feature can be part of N view-models. This maximizes testability, as we don't really need to test the view-models directly, because they will only declare initialization code, as we'll see in a bit, we focus our test efforts in the features declarations, and any view-model that uses that feature will receive an already tested method. It's a plug and play approach of view-models methods. 

Disclaimer. Here we use closures to make the communication between controller and view-model, but this approach of implementing view-models should not be affected by or affect whatever technique you pick to make this communication happen in your MVVM implementation.

## How features are declared
                 
```swift
// 1. Declare the protocol that defines the feature 
protocol FeatureOne: HasServiceOne & HasServiceTwo {}

// 2. Protocol extension with the method implementation (one by feature)
extension FeatureOne {
    func featureOne(completion: @escaping (Result<Void, Error>) -> Void) {        
        serviceOne.oneServiceCall(completion: completion)
    }
}
```

In a normal MVVM the featureOne method would be implemented inside a single view-model, which would kill reusability of the method in other parts of the code, and possibly lead to massive view models that contains many methods implementations. This is solved by the features composition, as it breaks up the view model in tiny parts that has single responsibilities and maximum reusability.

The features you declare may be dependent of other components, in the example above, serviceOne is a dependency for the featureOne to work. To make ServiceOne visible to the feature in question we declare that `FeatureOne` has service one through protocol inheritance, hence the name of the protocol `HasServiceOne`, which is defined in the `ServiceOne.swift` file, and declares that whoever implements/inherits from it needs to declare a serviceOne property of type `ServiceOneProtocol`. The `ServiceOne.swift` file would look like this: 

```swift
protocol HasServiceOne {
    var serviceOne: ServiceOneProtocol { get }
}

protocol ServiceOneProtocol {
    func oneServiceCall(completion: (Result<Void, Error>) -> Void)
}

final class ServiceOne: ServiceOneProtocol {
    func oneServiceCall(completion: (Result<Void, Error>) -> Void) {
        // call service and run the completion with the result
    }
}
```
All features dependencies should be declared via "Has" protocols, like analytics (HasAnalytics), dispatch queue (HasMainQueue, HasBackgroundQueue, HasXQueue), feature flags/toggles (HasFeatureFlag/HasFeatureToggle), application (HasApplication), device (HasDevice), etc. 

Features should never have dependencies on architectural components (controllers, view-models, coordinators, etc) as this would couple the features to specific screen architecture and go against the reusability purpose of this approach. But dependencies between architectural components can be kept as you may prefer. Check out bellow how the view model is injected in the controller in the "How controllers are declared" section of this post for a simple example. 
 
## How view-models are declared

```swift
// 1. Declare which features compose the view-model
typealias ViewModelFeatures = FeatureOne &
                              FeatureTwo

// 2. Declare the view-model concrete type
final class ViewModel: ViewModelFeatures {
    
    // 3. Dependencies needed by the features
    let serviceOne: ServiceOneProtocol
    let serviceTwo: ServiceTwoProtocol
        
    // 4. Dependency Injection
    init(
        serviceOne: ServiceOneProtocol,
        serviceTwo: ServiceTwoProtocol
     ) {
        self.serviceOne = serviceOne
        self.serviceTwo = serviceTwo
    }
}
```

Check how we don't need to declare any business logic in our view-model directly! As stated before, this is because our view-model is a composition of features, and only need to declare how the dependencies for those features are managed, and nothing else. How you manage the dependencies, simple or fancy, is up to you. 

Also, I would even consider making the view-model a struct, as it will make it even easier to be declared, just be sure you know the behinds the scenes and the consequences of these decision. If you don't, look up the [differences between struct and classes](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html), there is plenty of content in the internet. Following is an example of the ViewModel being implemented as a struct, as you see, it is shorter because we don't need to declare an initializer, as structs have memberwise initializers, based on its properties.

```swift
typealias ViewModelFeatures = FeatureOne &
                              FeatureTwo

struct ViewModel: ViewModelFeatures {
    let serviceOne: ServiceOneProtocol
    let serviceTwo: ServiceTwoProtocol
}
```
A four liner view-model, how about that?

## How controllers are declared

Not much to add here, just basic and simple view controller with the view-model being injected through the typealias that make the feature composition, that is declared in the same file as the view-model. This gives the controller visibility to the features that a view-model has and enable then to be used by the controller. 

```swift
final class ViewController: UIViewController {
    private let viewModel: ViewModelFeatures
    
    init(viewModel: ViewModelFeatures) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```
## How to unit test

As stated before, we will focus unit tests directly to our features instead of the view-models. If we test all the features correctly our view-models will also be tested.

The main difference for testing the features is that we can't actually instantiate a feature to run tests because it is a protocol, and not a concrete type. To solve that, we can simply implement a struct that implements the feature protocol to be tested. I recommend to declare this struct within the namespace of the test case itself, this helps not flooding autocompletion suggestions in your test target and help to keep things contained in their context.

Here follows how a test case would look like (full implementation in the sample source code): 

```swift
final class FeatureOneTestCase: XCTestCase {
    struct Sut: FeatureOne {
        let serviceOne: ServiceOneProtocol
    }

    private let serviceOneStub = ServiceOneStub()
    private lazy var sut = Sut(serviceOne: serviceOneStub)

    // Tests implementations goes here...
}
```

Even though there is a little bit more of boilerplate to the `XCTestCase`, having to declare the concrete implementation makes it immediately clear whats it's going to be tested, making it easier for other developers to understand whats going on in the test case.

For sake of completeness, here is the implementation of `ServiceOneStub`. If you are not familiar with stubs, take a look on what [test double](https://martinfowler.com/bliki/TestDouble.html) are. 

```swift
final class ServiceOneStub: ServiceOneProtocol {
    var response: Result<Void, Error> = .success(())
    
    func oneServiceCall(completion: (Result<Void, Error>) -> Void) {
        completion(response)
    }
}
```