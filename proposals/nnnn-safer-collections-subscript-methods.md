# Safer Collections subscript methods

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/nnnn-safer-collections-subscript-methods.md)
* Author(s): [Luis Henrique Borges](https://github.com/luish)
* Status: TBD
* Review manager: TBD

## Introduction

This proposal seeks to provide safer Collections [subscript](https://github.com/apple/swift/blob/7928140f798ae5b29af2053e774851f8012b555e/stdlib/public/core/Collection.swift#L147) 
methods for subranges in order to avoid
`index out of range` errors in execution time.

Swift-evolution thread: [link to the discussion thread for that proposal](http://thread.gmane.org/gmane.comp.lang.swift.evolution/14252)

## Motivation

Doing that in Swift causes a runtime error:

```swift
let a = [1,2,3]
let b = a[0..<5]
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

Considering that, the motivation is to provide a native and
handy interface to perform those operations in a simpler and safer way.
It would eliminate the need for validations, reduce the number of fatal errors
in runtime (if the validation was not effectively done by the user) 
and also simplify the code in cases where the subsequence length can be less 
than the range size.

## Proposed solution

The [mail list discussion](http://thread.gmane.org/gmane.comp.lang.swift.evolution/14252/focus=14382)
on the initial draft converged in a wider inclusion in the language that is worth considering. 
The proposed solution is to provide a convenient interface to let the user slice 
_collections_ implicit and explicitly through new labeled _subscript_ alternatives. 
These new subscript methods, described in more details below, would either truncate 
the range to the collection indices or return `nil` in cases where the range/index is 
out of bounds.

#### - subscript(`truncate` range: Range&lt;Index&gt;) -> SubSequence

The proposed solution is to clamp the range to the collection's bounds
before applying the subscript on it.

In the following example,

```swift
let a = [1,2,3]
let b = a[truncate: -1 ..< 5]
```

the range would be equivalent to `max(-1, a.startIndex) ..< min(5, a.endIndex)` 
which becomes `0 ..< 3` and `b` results in `[1,2,3]`.

#### - subscript(`safe` range: Range&lt;Index&gt;) -> SubSequence?

Returns `nil` whenever the range is out of bounds, 
instead of throwing a _fatal error_ in execution time. 

In the example below, `b` would be equal to `nil`.

```swift
let a = [1,2,3]
let b = a[safe: 0 ..< 5]
```

_* The label "safe" is just an idea, it might be improved as it
does not sound very appropriate in this context._

#### - subscript(`safe` index: Index) -> Element?

Similar behaviour as the previous method, but that subscripts given an _Index_ instead. 
Returns `nil` if the index is out of bounds. 

```swift
let a = [1,2,3]
let b = a[safe: 5] // nil
```

This behaviour could be considered consistent with dictionaries, other 
collection type in which the _subscript_ function returns `nil` if the 
dictionary does not contain the key given by the user.

In summary, considering `a = [1,2,3]`:

- `a[0 ..< 5]` results in _fatal error_, the current implementation (_fail fast_).
- `a[truncate: 0 ..< 5]` turns into `a[truncate: 0 ..< 3]` and produces `[1,2,3]`.
- `a[safe: 0 ... 5]` returns `nil` indicating that the range is invalid, but not throwing any error.
- `a[safe: 3]` also returns `nil`, as the valid range is `0 ..< 3`.

## Detailed design

This is a simple implementation for the _subscript_ methods I am proposing:

```swift
extension CollectionType where Index: Comparable {
    
    subscript(truncate range: Range<Index>) -> SubSequence {
        let start = max(startIndex, range.startIndex)
        let end = min(endIndex, range.endIndex)
        return self[start ..< end]
    }
    
    subscript(safe range: Range<Index>) -> SubSequence? {
        guard range.startIndex >= startIndex && range.endIndex <= endIndex
            else { return nil }
        return self[range]
    }
    
    subscript(safe index: Index) -> Generator.Element? {
        guard index >= startIndex && index < endIndex
            else { return nil }
        return self[index]
    }
    
}
```

Examples:

```swift
let a = [1, 2, 3]
    
a[truncate: 0 ..< 5] // [1, 2, 3]
a[truncate: -1 ..< 2] // [1, 2]
a[truncate: 1 ..< 2] // [2]
a[truncate: 3 ..< 4] // []
a[truncate: 4 ..< 3] // Fatal error: end < start

a[safe: -1 ..< 5] // nil
a[safe: -1 ..< 2] // nil
a[safe: 0 ..< 5] // nil
a[safe: 1 ..< 3] // [2, 3]
a[safe: 4 ..< 3] // Fatal error: end < start

a[safe: 0] // 1
a[safe: -1] // nil
a[safe: 3] // nil
```

## Impact on existing code

It does not cause any impact on existing code, the current 
behaviour will continue as the default implementation.

## Alternatives considered

An alternative would be to make the current subscript method `Throwable`
motivated by this blog post published by @erica:
[Swift: Why Try and Catch don’t work the way you expect](http://ericasadun.com/2015/06/09/swift-why-try-and-catch-dont-work-the-way-you-expect/)
