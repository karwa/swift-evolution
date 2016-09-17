# Super-powered protocols (not ready obviously)

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Karl](https://github.com/swiftdev), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

Protocols are fundamental to Swift; they allow us to represent different abstractions for the same data. Unfortunately, they do not overlap gracefully which limits our ability to use them as a modelling tool.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

If a type is declared as conforming to a protocol, members of that type are matched against the requirements of the protocol. However, protocols do not impose any requirements about the conforming types data - functions do not, and properties may be stored or computed. It is not wrong to say that a Swift protocol is a 'view' of looking at some data, complete with not just compiler- but also semantic guarantees.

Currently the requirements of a protocol are matched against the direct members of the type. This means that protocols which may be conformed to by the same type must be careful to avoid using the same names to represent the same concept. Where a conflict does arise, and a requirement exists in both protocols with the same name but different semantics, it must choose between conforming to one or the other. The other choice is to create a wrapper structure which conforms to the protocol; for example, `String` provides many views of its data (UTF8/16/Scalars/Characters) which must all conform to Collection in different ways.

As a modelling tool, it is clear that protocols are limited in their expressiveness.

## Proposed solution


My proposed solution is for protocol conformances to be explicit for all members, and for the protocol to be embedded in the function's mangled name (ABI). At a high level, this means that rather than a type owning its member directly, the type owns a conformance which owns the member:

```swift
protocol MyProtocol {
    var aVariable : Int { get set }
    func doSomething()
}

protocol AnotherProtocol {
    var aVariable : Int { get set }
    func doSomething()
}

class MyClass : MyProtocol {
    var MyProtocol.aVariable = 42
    func MyProtocol.doSomething() { /*...*/ }
}

class MyDerivedClass : MyClass, AnotherProtocol {
    var AnotherProtocol.aVariable = 36
    func AnotherProtocol.doSomething() { /*...*/ }
}

let anInstance = MyDerivedClass()
anInstance.MyProtocol.aVariable = 18 // Type -> Conformance -> Member

assert(anInstance.MyProtocol.aVariable != anInstance.AnotherProtocol.aVariable) // true
```

The compiler synthesizes a getter with the name of the protocol (TODO: should we always lowercase the first letter?). The return value of this getter is the protocol witness:

```swift
    var MyProtocol : MyClass.MyProtocol { get }
```

One nice feature would be if this property were overridable, so that types which conform to a protocol may delegate to an optimised wrapper. This wrapper would have to meet certain requirements, such as preserving mutation guarantees for mutating members (needs documentation). Protocols do not impose a particular layout for your data, but they may inform how it is intended to be consumed and being able to manually optimise based on that information is important.

Now, the fully-qualified syntax shown above is likely to become annoying in the common case of members which do not conflict with other protocols. Nobody wants to have to write `[1, 2, 3].Collection.count` every time. So for unique members, we are able to deliver a convenient shorthand.

```swift
let baseInstance = MyClass()
baseInstance.aVariable = 12 // Compiler will expand this out to baseInstance.MyProtocol.aVariable
```

Since we do not allow conflicting member names at the moment, this shorthand would limit source-breaking changes to the declaration site, where the same rule means migration could be performed with a simple fix-it. However, since it is a shorthand, if you add an extension to Array which also defines `count` (or conforms to a protocol which does), the name will become ambiguous for the visibility of that extension.

Retroactive modelling presents a challenge. What if we shipped version 1 of MyClass without MyProtocol conformance, and some refactoring in version 1.1 presented the opportunity to declare a protocol? In this case, the implementations of aVariable's getters and setters and of doSomething will have moved, but we have also added some semantic meaning to them. We need to verify that we haven't changed the members semantics, and provide an opportunity to support both semantics if required. Since the type-level names are now vacated, we can implement all of these things:

```swift
// v1.0
class MyClass { /* as above */ }

// v1.1
protocol MyProtocol { /* as above */ }

class MyClass : MyProtocol {

    @makeAvailable(MyProtocol, as: _, 1.0) // compiler should generate jumps in '_' (MyClass) for all members of MyProtocol to support version API version 1.0

    var MyProtocol.aVariable = 42
    func MyProtocol.doSomething() { /*...*/ }
}
```

Note: this is different to the shorthand syntax. The shorthand syntax is expanded by the compiler to include the full protocol name where it can exactly identify the intended protocol.

The same approach (of generating jumps to the new locations) is to be used to support renamed protocols. Developers should be warned that renaming public protocols very often whilst maintaining binary compatibility will inflate their executable sizes for each type which conforms to the protocol.

Now it is possible to implement the same protocol in multiple ways. For example, all of String's views could be made in to protocols inheriting from Collection:

```swift
// These are protocols because they have semantic meaning - a collection of bytes != a collection of UTF8
protocol UTF8Stream : Streamable          { typealias Element = UTF8CodeUnit ... }
protocol UTF16Stream : Streamable         { typealias Element = UTF16CodeUnit ... }
protocol UnicodeScalarStream : Streamable { typealias Element = UnicodeScalarCodeUnit ... }

extension String : UTF8Stream {
    func UTF8Stream.next() -> UTF8CodeUnit {
    // transcode next character to UTF8
    }
}

extension String : UTF16Stream {
    func UTF16Stream.next() -> UTF16CodeUnit {
    // transcode next character to UTF16, if neccessary
    }
}

..etc
```

The current solution, involving structs for each view, produces a syntax not dissimilar to as it would be under this proposal:

```swift
// current
"testing".utf8View.first!

// under this proposal
"testing".UTF8Collection.first!
// or even better...
let firstChar : UTF8CodeUnit = "testing".first! // unambiguous reference to String.UTF8Collection.first
```

The latter syntax would make up for the loss of character literals, which are really annoying when interacting with C APIs.

## Detailed design

At a high level, I see the implementation work going like this:
- Parser: Require explicitly declaring protocol for all members which conform to a protocol
- Compiler: Use that information to build protocol witnesses.
- Compiler: Synthesise getter for “.ProtocolName”, which returns the protocol witness
- (Optional) Make overridable to supply your own witness for a protocol
- Thunk-generation syntax for retroactive modelling and protocol renaming

## Impact on existing code

Despite the somewhat radical nature of the proposal, we can keep source-breaking changes to the declaration site, and those can be automatically migrated.


## Alternatives considered

- Do nothing. Protocols must be careful never to step on the toes of any future future protocol which anybody may want to use together with themselves. That seems like a bit of a difficult limitation to live with for Swift as such a protocol-oriented language.
- How else could we disambiguate protocol references?
