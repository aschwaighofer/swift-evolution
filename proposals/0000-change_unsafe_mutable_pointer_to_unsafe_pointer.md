# Replace `UnsafeMutablePointer` by `UnsafePointer` in non-mutating APIs

* Proposal: [SE-0000](0000-change_unsafe_mutable_pointer_to_unsafe_pointer.md)
* Author: [Arnold Schwaighofer](https://github.com/aschwaighofer)
* Status: **Pitch**
* Review manager: TBD

## Introduction

`UnsafeMutablePointer` didn't always have an immutable variant, and when it was
introduced there could have been many places that should have been changed to
use it, but were not. `UnsafeMutablePointer` is still pervasive. We should
survey the uses and make sure they're all using mutability only as appropriate.

The only such occurrence I found was the `dprintf` API which should have a
read-only format argument.

- Swift-evolution thread: [Pitch](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160718/024780.html)
- Swift bug: [SR-1958] (https://bugs.swift.org/browse/SR-1958)
- Branch with changes to stdlib: [unsafe_mutable_pointer_to_unsafe_pointer] (https://github.com/aschwaighofer/swift/tree/unsafe_mutable_pointer_to_unsafe_pointer)

## Motivation

We should change uses of `UnsafeMutablePointer` with `UnsafePointer` to clearly
express the intent of the API where appropriate.

## Proposed solution

The proposed solution is to change `dprintf` API to take a `UnsafePointer`
argument.

## Detailed design

Change:
```swift
public func dprintf(_ fd: Int, _ format: UnsafeMutablePointer<Int8>, _ args: CVarArg...) -> Int32
```

To
```swift
public func dprintf(_ fd: Int, _ format: UnsafePointer<Int8>, _ args: CVarArg...) -> Int32
```
## Impact on existing code

Existing coersions to `UnsafePointer` from `UnsafeMutablePointer` will allow
existing code to work.

## Alternatives considered

Leave as is.


