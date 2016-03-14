# Add an API to `Unmanaged` to get the instance it holds `@guaranteed`

* Proposal: [SE-0000 Add an API to Unmanaged to get the instance it holds guaranteed](https://github.com/aschwaighofer/swift-evolution/blob/unmanaged_unsafe_guaranteed_proposal/proposals/0000-unsafe-guaranteed.md)
* Author(s): [Arnold Schwaighofer] (https://github.com/aschwaighofer)
* Status: **Awaiting Review**
* Review manager: TBD

## Introduction

The standard library `Unmanged<Instance>` struct provides an instance wrapper
that does not participate in ARC; it allows the user to make manual
retain/release calls and get at the contained instance at balanced/unbalanced
retain counts.

This proposal suggests to add another method to `Unmanaged` that will allow
making an assertion about the guaranteed lifetime of the instance returned.

This assertion will help the compiler remove ARC operations.

``` swift
public struct Unmanaged<Instance : AnyObject> {

  // Get the value of the unmanaged referenced as a managed reference without
  // consuming an unbalanced retain of it. Asserts that some other owner
  // guarantees the lifetime of the value for the lifetime of the return managed
  // reference. You are responsible for making sure that this assertion holds
  // true.
  //
  // NOTE:
  // Be aware that storing the returned managed reference in an L-Value extends
  // the lifetime this assertion must hold true for by the lifetime of the
  // L-Value.
  //   var owningReference = Instance()
  //   var lValue : Instance
  //   withFixedLifetime(owningReference) {
  //     lValue = Unmanaged.passUnretained(owningReference).takeGuaranteedValue()
  //   }
  //   lValue.doSomething() // Bug: owningReference lifetime has ended earlier.

  public func takeGuaranteedValue() -> Instance {
    let result = _value
    return Builtin.unsafeGuaranteed(result)
  }
```

Prototype: [link to a prototype implementation](https://github.com/aschwaighofer/swift/tree/unsafe_guaranteed_prototype)

## Motivation

A user can often make assertions that an instance is kept alive by another
reference to it for the duration of some scope.

Consider the following example:

```swift
class Owned {
  func doSomeWork() {}
}

public class Owner {
  var ref: Owned

  init() { ref = Owned() }

  public func doWork() {
    doSomething(ref)
  }

  func doSomething(o: Owned) {
     o.doSomeWork()
  }
}
```

In this context the lifetime of `Owner` always exceeds `o: Ownee`. However,
there is currently no way to tell the compiler about such an assertion.  We can
pass reference counted values in an `Unmanged` container without incurring
reference count changes.

```swift
public class Owner {

  public func doWork() {
    // No reference count for passing the parameter.
    doSomething(Unmanaged.passUnretained(ref))
  }

  func doSomething(u: Unmanaged<Owned>) {
    // ...
  }
}
```

We can get at the contained instance by means of ``takeUnretainedValue()`` which
will return a balanced retained value: the value is returned at `+1` for release
by the caller. However, when it comes to accessing the contained instance we
incur reference counting.

```swift
  func doSomething(u: Unmanaged<Owned>) {
    // Incurs refcount increment before the call and decrement after for self.
    u.takeUnretainedValue().doSomeWork()
  }
```

With the proposed API call the user could make the assertion that `u`'s
contained instance's lifetime is guaranteed by another reference to it.

```swift
  func doSomething(u: Unmanaged<Owned>) {
    // Incurs refcount increment before the call and decrement after for self
    // that can be removed by the compiler based on the assertion made.
    u.takeGuaranteedValue().doSomeWork()
  }
```

The compiler can easily remove the reference counts marked by this method call
with local reasoning.

## Impact on existing code

This is a new API that does not replace an existing method. No existing code
should be affected.

## Alternatives considered

A somewhat related proposal would be to allow for class types to completely opt
out of ARC forcing the programmer to perform manual reference counting. The
scope of such a proposal would be much bigger making it questionable for Swift 3
inclusion. I believe that the two approaches are complementary and don't
completely supplant each other. This proposal's approach allows for selectively
opting out of ARC while providing a high notational burdon when wide spread over
the application (wrapping and unwrapping of `Unmanaged`). Opting completely out
of ARC is an all-in solution, which on the other hand does not have the high
notational burden.
