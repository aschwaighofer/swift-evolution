# Add an API to `Unmanaged` to get the instance it holds `@guaranteed`

* Proposal: [SE-0000 Add an API to Unmanaged to get the instance it holds guaranteed](https://github.com/aschwaighofer/swift-evolution/blob/unmanaged_unsafe_guaranteed_proposal/proposals/0000-unsafe-guaranteed.md)
* Author(s): [Arnold Schwaighofer] (https://github.com/aschwaighofer)
* Status: **Draft**
* Review manager: TBD

## Introduction

The standard library `Unmanged<Instance>` struct provides an instance wrapper
that does not participate in ARC; it allows the user to make manual
retain/release calls and get at the contained instance at balanced/unbalanced
retain counts.

This proposal suggests to add another method `withUnsafeGuaranteedValue` to
`Unmanaged` that accepts a closure. Calling this method is akin to making an
assertion about the guaranteed lifetime of the instance for the
delinated scope of the method invocation.

```swift
  func doSomething(u : Unmanged<Owned>) {
    // The programmer asserts that there exists another reference to the
    // unmanaged reference stored in 'u' and that the lifetime of the referenced
    // instance is guaranteed to extend beyond the 'withUnsafeGuaranteedValue'
    // invocation.
    u.withUnsafeGuaranteedValue {
      $0.doSomething()
    }
  }
```

This assertion will help the compiler remove ARC operations.

``` swift
public struct Unmanaged<Instance : AnyObject> {

  // Get the value of the unmanaged referenced as a managed reference without
  // consuming an unbalanced retain of it and pass it to the closure. Asserts
  // that there is some other reference ('the owning reference') to the
  // unmanaged reference that guarantees the lifetime of the unmanaged reference
  // for the duration of the 'withUnsafeGuaranteedValue' call.
  //
  // NOTE: You are responsible for ensuring this by making the owning reference's
  // lifetime fixed for the duration of the 'withUnsafeGuaranteedValue' call.
  //
  // Violation of this will incur undefined behavior.
  //
  // A lifetime of a reference 'the instance' is fixed over a point in the
  // programm if:
  //
  // * There is another managed reference to the same instance 'the instance'
  //   whose life time is fixed over the point in the program by means of
  //  'withExtendedLifetime' closing over this point.
  //
  //   var owningReference = Instance()
  //   ...
  //   withExtendedLifetime(owningReference) {
  //       point($0)
  //   }
  //
  // Or if:
  //
  // * There is a class, or struct instance ('owner') whose lifetime is fixed at
  //   the point and which has a stored property that references 'the instance'
  //   for the duration of the fixed lifetime of the 'owner'.
  //
  //  class Owned {
  //  }
  //
  //  class Owner {
  //    final var owned : Owned
  //
  //    func foo() {
  //        withExtendedLifetime(self) {
  //            doSomething(...)
  //        } // Assuming: No stores to owned occur for the dynamic lifetime of
  //          //           the withExtendedLifetime invocation.
  //    }
  //
  //    func doSomething() {
  //       // both 'self' and 'owned''s lifetime is fixed over this point.
  //       point(self, owned)
  //    }
  //  }
  //
  // The last rule applies transitively through a chains of stored references
  // and nested structs.
  //
  // Examples:
  //
  //   var owningReference = Instance()
  //   ...
  //   withExtendedLifetime(owningReference) {
  //     let u = Unmanaged.passUnretained(owningReference)
  //     for i in 0 ..< 100 {
  //       u.withUnsafeGuaranteedValue {
  //         $0.doSomething()
  //       }
  //     }
  //   }
  //
  //  class Owner {
  //    final var owned : Owned
  //
  //    func foo() {
  //        withExtendedLifetime(self) {
  //            doSomething(Unmanaged.passUnretained(owned))
  //        }
  //    }
  //
  //    func doSomething(u : Unmanged<Owned>) {
  //      u.withUnsafeGuaranteedValue {
  //        $0.doSomething()
  //      }
  //    }
  //  }
  public func withUnsafeGuaranteedValue<Result>(
    @noescape closure: (Instance) throws -> Result
  ) rethrows {
    let instance = _value
    let (guaranteedInstance, token) = Builtin.unsafeGuaranteed(instance)
    try closure(guaranteedInstance)
    Builtin.unsafeGuaranteedEnd(token)
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
    withExtendedLifetime(self) {
      doSomething(ref)
    }
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
    u.withUnsafeGuaranteedValue {
      $0.doSomeWork()
    }
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
