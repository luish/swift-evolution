# Safer half-open range operator

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/0065-array-slicing-open-range.md)
* Author(s): [Luis Henrique Borges](https://github.com/luish)
* Status: TBD
* Review manager: TBD

## Introduction

This proposal seeks to provide a safer `..<`
(aka [half-open range operator](https://github.com/apple/swift/blob/510f29abf77e202780c11d5f6c7449313c819030/stdlib/public/core/Range.swift#L193))
in order to avoid `Array index out of range` errors during the execution time.

Swift-evolution thread: [link to the discussion thread for that proposal](http://thread.gmane.org/gmane.comp.lang.swift.evolution/14252)

## Motivation

Doing that in Swift causes a runtime error:

```swift
let a = [1,2,3]
let b = a[0..<5]
print(b)
```

```
> Error running code:
> fatal error: Array index out of range
```

In comparison with other languages (often referred to as
"modern languages"), we see the exact behavior I am
going after in this proposal.

Python:

```python
>>> a = [1,2,3]
>>> a[:5]
[1, 2, 3]
```

Ruby:

```ruby
> a = [1,2,3]
> a[0...5]
=> [1, 2, 3]
```

Considering that, the motivation is to avoid validations
on the array size before slicing it, using purely:

```swift
b = a[0..<5]
```

instead of:

```swift
if a.count > 5 {
  b = a[0..<5]
}
```

## Proposed solution

The proposed solution is to slice the array returning all elements that are
below the half-open operator, even though the number of elements is lesser
than the ending of the range.

```swift
let a = [1,2,3]
let b = a[0..<5]
print(b)

> [1,2,3]
```

This would eliminate the need for verifications on the array size before
slicing it -- and consequently runtime errors in cases when the programmer didn't.

## Detailed design

It does not require any change in the language syntax. It is all about
the internal behavior of the existing implementation for the operator
when applied to arrays.

## Impact on existing code

It does not cause any impact on existing code.

## Alternatives considered

As [proposed by Haravikk and Vladimir in the mail list thread](http://thread.gmane.org/gmane.comp.lang.swift.evolution/14252/focus=14253), perhaps a variation of the operator would be a better approach, making the user responsible for deciding whether to use the operator implicit or explicitly, just like [Overflow Operators](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AdvancedOperators.html#//apple_ref/doc/uid/TP40014097-CH27-ID37) do.

Considering a new operator, let's say `&..<` (aka _'safer half-open operator'_), the following statement

```swift
let a = [1,2,3]
let b = a[-1 &..< 5]
```

would be equivalent to `let b = a[max(0, a.initialIndex) ..< min(5, a.endIndex)]`, which becomes `let b = a[0 ..< 3]` and does not raise any error or throw any exception in execution time.

Another alternative would be to make this operator `Throwable`
motivated by this blog post published by @erica:
[Swift: Why Try and Catch don’t work the way you expect](http://ericasadun.com/2015/06/09/swift-why-try-and-catch-dont-work-the-way-you-expect/)
