---
layout: page
title: singletons
---

```swift

import UIKit

//: ## What is a singleton

class MySingletonClass {
    static let singleton = MySingletonClass() //1.2 syntax
    
    struct ArbitraryStruct { //1.1 syntax
        static let singleton = MySingletonClass()
    }
}

MySingletonClass.singleton

//: ## Singleton as global variables
//: It turns out that singletons are pretty much like global variables:
//: 
//: 1. Accessible from everywhere
//: 2. State may "come from" anywhere and be used by anywhere
//: 3.  Is there really any difference between

MySingletonClass.singleton

// and

let GlobalVariable = MySingletonClass()

//: ## Singletons as real life
//:
//: In the real world, we have singletons.
//:
//: 1.  You only have one iPhone
//: 2.  There's only one internet
//: 3.  There's only one GPS
//: 4.  There's only one display
//:
//: Practical singletonology is about figuring out how to handle the **border condition** between "singleton world" and "instance world".  
//: Even if *you* never create a singleton, they still exist, because your app runs in the real world and we only have one display.

//: ## Or do we?
//:
//: 1.  We now have external displays in iOS.  You're reading this on one.
//: 2.  Apple watch is internally an external display for iOS
//: 3.  You may (read: **should**) have unit tests that create UIViews that don't exist on any display
//:
//: Beware the singletons that aren't quite.  I call them "snigletons".



//: ### Snigleton #1: unit test knock-off

// We have one instance for the app to use

MySingletonClass.singleton

//and another one for the unit tests

import XCTest
class MyTest : XCTestCase {
    func testFoo() {
        let secondInstance = MySingletonClass() //now we have 2 singletons!
    }
}

//: The following is real code in production

class MySingleton1 {
    static let singleton = MySingleton1()
    init(for_unit_testing_only_i_know_what_im_doing : Bool = false) {
        if for_unit_testing_only_i_know_what_im_doing {
            //snip
        }
        else {
            //snip
        }
    }
}

import XCTest
class MyTest2 : XCTestCase {
    func testFoo() {
        let secondInstance = MySingleton1(for_unit_testing_only_i_know_what_im_doing: true)//now we have 2 singletons!
    }
}

//: ### Snigleton #2: The disappearing instance
//: The insight here is that singletons can accumulate state, and sometimes that state needs to get blown away.
//:
//: Examples:
//:
//: 1.  Caching
//: 2.  Unit test isolation
//: 3.  Document/user isolation


class Snigleton2 {
    static private var _singleton : Snigleton2? = nil
    static var singleton : Snigleton2 {
        get {
            if _singleton == nil {
                _singleton = Snigleton2()
            }
            return _singleton!
        }
    }
    
    static func die() {
        _singleton = nil
    }
}

//: ### Snigleton #3: The Doubleton

class Snigleton3 {
    let firstInstance = Snigleton3()
    let secondInstance = Snigleton3()
}

//: Examples
//: 
//: 1.  You're writing pong, which has exactly 2 players
//: 2.  You're doing audio, and users have exactly 2 ears
//: 3.  You're talking to the database, either from the foreground or the background

//: ### Snigleton #4: The proxy

class Snigleton4 {
    private static let instances : [Snigleton4] = [Snigleton4(), Snigleton4(), Snigleton4(), Snigleton4()]
    
    func foo() { }
    
    static func foo() {
        let randomIndex = Int(arc4random_uniform(UInt32(instances.count)))
        instances[randomIndex].foo()
    }
}

Snigleton4.foo()
//: Examples
//:
//: 1.  Load balancing
//: 2.  Multithreading
//: 3.  Distributed networking

//: ## Lifecycle of a singleton
//:
//: Most singletons tend to go through 3 life stages
//: 
//: ### Singleton
//:
//: You're in a design meeting, and somebody tells you "We only support a single user".  So you write a singleton, it works, you check it in, you go home.
//:
//: ### Snigleton
//:
//: It turns out that in reality you want unit tests that involve multiple users (unit test snigleton), or that users can log out (disappearing snigleton), so your singleton becomes a snigleton.
//:
//: ### Instance
//:
//: Feature request: support multiple users
//:
//: So you throw up your hands and reimplement it.
//:
//: ## Conclusion
//: You will waste less code if you can skip some of the steps.  Ask yourself:
//: 1.  How confident are we that our application supports only 1 user? Will we change our mind later? (Skip from singleton to instance)
//: 2.  Is this really a singleton, or do one of the snigleton patterns fit better?  (Skip from singleton to snigleton)
//: 3.  Have I thought about how I want to test this?
//: 4.  Have I thought about accumulation of state and do I need to reset it?
//: 5.  Have I thought about the Apple Watch (display is not a singleton)?  Have I thought about iOS extensions?  (PID is not a singleton)?  Have I considered that Apple reliably announces stuff like this once a year?

```