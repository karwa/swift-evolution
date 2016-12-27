# Change (Dispatch)Data.`withUnsafeBytes` to use `UnsafeMutableBufferPointer`

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Karl Wagner](https://github.com/karwa
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

The standard library's `Array` and `ContiguousArray` types expose the method `withUnsafeBytes`, which allows you to view their contents as a contiguous collection of bytes. The core libraries Foundation and Dispatch contain types which wrap some allocated data, but their `withUnsafeBytes` method only allows you to view the contents as a pointer to a contiguous memory location of a given type.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

The current situation makes it awkward to write generic code. Personally, I use the following extension in my projects to sort the naming confusion out:

```swift
protocol UnsafeRawBytesConvertible {
  // Create yet another name.
  func withUnsafeRawBytes<T>(execute: (UnsafeRawBufferPointer) throws -> T) rethrows -> T
}

// Array is fine, just forward to the new name.
extension Array : UnsafeRawBytesConvertible {
  func withUnsafeRawBytes<T>(execute: (UnsafeRawBufferPointer) throws -> T) rethrows -> T {
    return try withUnsafeBytes(execute)
  }
}

#if canImport(Dispatch)
  import Dispatch

  // DispatchData only gives us a pointer, we need pointer + count.
  extension DispatchData : UnsafeRawBytesConvertible {
    func withUnsafeRawBytes<T>(execute: (UnsafeRawBufferPointer) throws -> T) rethrows -> T {
      return try withUnsafeBytes { try execute(UnsafeRawBufferPointer(start: $0, count: count)) }
    }
  }
#endif

#if canImport(Foundation)
  import Foundation

  // Data only gives us a pointer, we need pointer + count.
  extension Data : UnsafeRawBytesConvertible {
    func withUnsafeRawBytes<T>(execute: (UnsafeRawBufferPointer) throws -> T) rethrows -> T {
      return try withUnsafeBytes { try execute(UnsafeRawBufferPointer(start: $0, count: count)) }
    }
  }
#endif
```

Conceptually, the corelibs types _are_ untyped regions of memory, and it would make sense for them to adopt the `UnsafeRawBufferPointer` model.

## Proposed solution

The proposed solution would be to deprecate the current methods on (Dispatch)Data (with 2 generic parameters), and replace them with methods with identical signatures to Array (with 1 generic parameter).

To be deprecated:
```swift
public func withUnsafeBytes<ResultType, ContentType>(_ body: (UnsafePointer<ContentType>) throws -> ResultType) rethrows -> ResultType
```

Replaced with:
```swift
public func withUnsafeBytes<R>(_ body: (UnsafeRawBufferPointer) throws -> R) rethrows -> R
```

## Source compatibility

Source-breaking. Users binding a (Dispatch)Data to an UnsafePointer<T> would instead have to call:
```swift
pointer.baseAddress!.assumingMemoryBound(to: T.self)
```

## Effect on ABI stability

Source-breaking change to corelibs APIs.

## Effect on API resilience

Source-breaking change to corelibs APIs.

## Alternatives considered

- A different method on `Data` and `DispatchData`, providing an `UnsafeRawBufferPointer`? There would still be a naming discrepency between the stdlib and corelibs types
