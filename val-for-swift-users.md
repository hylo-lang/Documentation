# A primer on the design of Val

[Val](https://www.val-lang.dev) is a Swift-inspired research language that @dabrahams and I are developing with a handful of other people.
We think we may contribute some ideas back to Swift, so we thought it might be useful to gather them in a dedicated post, rather than sprinkling them in various responses across the forum.

This post has not for objective to sell Val to anyone.
Its purpose is to present some of Val's ideas through the lens of Swift to understand whether they could make sense in the latter.

Significant portions of this post synthesize parts of [Val's language tour](https://www.val-lang.dev/pages/language-tour.html), with additional comments describing how Val relates to Swift.

## No reference semantics

Val started with one question:
*What if you have nothing but [mutable value semantics](https://www.jot.fm/issues/issue_2022_02/article2.pdf) in a programming language?*

So we took Swift, removed classes, and went from there.
Val only has value types (i.e., Swift's structs).
It does not allow global mutable variables and its closures have value semantics too (note: closures have reference semantics in Swift).

## Explicit copies

Another question we wanted to answer with Val is:
*How can we deal with non-copyable types in a language with strong support for generic programming?*

So we made all types non-copyable by default.
We also wanted to avoid having code behave in a different way depending on whether things are copyable.
We beleive that's an important for generic code, so that functions such as the one below have a uniform behavior, regardless of their instantiation.

```
fun assign<T>(value: T, to target: inout T) {
  target = value // won't work if `value` is not copyable
}
```

So we decided that all copies should be explicit.
Though it may seem like a drastic and inconvenient design choice, it turns out it's actually a good bet because:

- Passing arguments by value doesn't require copying. In fact, that's what Swift already acknowledges under the cover with the `@guaranteed` passing convention in SIL.
- Local `let` bindings can behave exactly like arguments passed by value.
- Programming languages with implicit copying (e.g., Swift, C++, Rust) are typically too eager to copy values.

To illustrate this point, we can look at the method below.
It partitions a collection in-place so that elements on the left and right of a pivot are respectively smaller and greater than that pivot.

```
extension BidirectionalCollection
where Self: MutableCollection, Element: Comparable
{

  fun partition() inout -> Optional<Index> {
    if is_empty() { return nil }

    let pivot = index(before: end_index())
    var i = start_index()
    var j = start_index()

    while j != index(before: end_index()) {
      if self[j] < self[pivot] {
        &swap_at(i, j)
        i = index(after: i)
      }
      j = index(after: j)
    }

    &swap_at(i, j)
    return i
  }

}
```

*Note: `&` is simply a marker required by the language when a value is mutated, just like it is a marker for arguments passed to `inout` parameters in Swift.*

Notice that, unlike in Swift, `start_index` is a method rather than a property.
The reason is that, in Val, functions and methods *return* values while a properties and subscripts *project* one.
So functions create independent values that can be used just as linear (i.e. non-copyable objects).
As a result, `partition` does not require a single explicit copy.
We are using all values linearly.

The compiler can even detect eager copies inserted by the user.

```
fun unused_space(_ things: Array<String>) -> Int {
  let space = things.capacity() - things.count()
  return space.copy() // warning: remove unnecessary copy
}
```

In the example above, there is no need to copy `space` since it is an independent value that is not used after the return statement.
Thefore, the compiler can suggest to remove that unnecessary call to `copy` as a fix-it.

Conversely, the compiler can detect when a copy is needed.
For example:

```
type Vector2 {
  var x: Double
  var y: Double

  /// Initializes a 2D vector from the value of its components.
  init(_ buffer: Double[2]) {
    x = buffer[0].copy()
    y = buffer[1] // error: cannot let `buffer[1]` escape, insert a copy
  }
}
```

Because the compiler can effectively know when a copy would have been inserted in a language with implicit copying, we can let it do it on its own in specific scopes.
That results in a behavior that mimics what Swift does.
Here, for example, we're using a directive in the scope of `init(_:)` to let the compiler insert the appropriate copies on its own.

```
init(_ buffer: Double[2]) {
  @implicitcopy
  x = buffer[0]
  y = buffer[1]
}
```

*Note: one may parameterize `@implicitcopy` by a trait so that it only applies to specific types.*

To make a type copyable, one makes it conform to the `Copyable` trait.
A trait is like a protocol in Swift.

```
type Vector2: Copyable {
  var x: Double
  var y: Double
}
```

Just like in Swift, requirements of traits like Copyable, Equatable and Hashable can be synthesized by assuming that all stored properties form whole/part relationships.

`Copyable` is defined as follows in the standard library:

```
trait Copyable: Sinkable {

  // Creates a copy of `self`
  fun copy() -> Self

  // Writes a copy of `self` into `other`.
  fun copy(into other: inout Self) {
    other = self.copy()
  }

}
```

The first method must be implemented (or synthesized) for all copyable types.
The second method can be used to optimize assignments.
When a assignment occurs, the LHS is deinitialized and re-initialized with the value of the RHS.
However, there are situations where the LHS's storage could be reused, thus avoiding unnecessary memory allocation.
So the compiler automatically transforms assignments of the form `a = b.copy()` into `b.copy(into: &a)`.

`Copyable` is a refinement of `Sinkable`, which denotes movable types.

## Parameter passing conventions

A parameter passing convention describes how an argument is passed from caller to callee.
In Val, there are four: `let`, `inout`, `sink` and `set`.
The next series of example define four corresponding functions to offset this 2-dimensional vector type:

```
type Vector2: Copyable {
  var x: Double
  var y: Double
}
```

### `let` parameters

`let` is the default convention and does not need to be spelled explicitly.
At the user level, `let` parameters can be understood just like reglar parameters in Swift.
They are (notionally) passed by value, and are truly immutable.

```
fun offset_let(_ v: Vector2, by delta: Vector2) -> Vector2 {
  Vector2(x: v.x + delta.x, y: v.y + delta.y)
}
```

The `let` convention does not transfer ownership of the argument to the callee, meaning, for example, that without first copying it, a `let` parameter can't be returned, or stored anywhere that outlives the call.

```
fun duplicate(_ v: Vector2) -> Vector2 {
  v // error: `v` cannot escape; return `v.copy()` instead.
}
```

### `inout` parameters

`inout` behaves almost exactly like in Swift.
Arguments to `inout` parameters must be unique, mutable, and marked with an ampersand (`&`) at the call site.

```
fun offset_inout(_ target: inout Vector2, by delta: Vector2) {
  &target.x += delta.x
  &target.y += delta.y
}
```

One difference with Swift is that, although `inout` parameters are required to be valid at function entry and exit, a callee is entitled to do anything with the value of such parameters, including destroying them, as long as it puts a value back before returning.
That is convenient to temporarily move an argument into an object that encapsulates some computation.

```
type Processor {
  var ast: AST
  fun process() inout { ... }
  fun process_expr() inout { ... }
  fun process_decl() inout { ... }
  fun finalize() sink -> {AST, Result} { ... }
}

fun process(_ ast: inout AST) -> Result {
  // move `ast` into `p`, without copying it
  var p = Processor(ast)
  
  // Execute the computation encapsulated by `Processor`
  &p.process()

  // Move `p` back into `ast`.
  let result: Result
  (ast, result) = p.finalize()
  return result
}
```

*Note: The convention of `self` in a method is written after the parameter list in Val.*

### `sink` parameters

The `sink` convention indicates a transfer of ownership, so unlike previous examples the parameter can escape the lifetime of the callee.

```
fun offset_sink(_ base: sink Vector2, by delta: Vector2) -> Vector2 {
  &base.x += delta.x
  &base.y += delta.y
  return base        // OK; base escapes here!
}
```

Just as with `inout` parameters, the compiler enforces that arguments to `sink` parameters are unique.
Because of the transfer of ownership, though, the argument becomes inaccessible in the caller after the callee is invoked.

```
fun main()
  let v = Vector2(x: 1, y: 2)
  print(offset_sink(v, (x: 3, y: 5)))  // prints (x: 4, y: 7)
  print(v) // <== error: v was consumed in the previous line
}          // to use v here, pass v.copy() to offset_sink.
```

The `sink` and `inout` conventions are closely related; so much so that `offset_sink` can be written in terms of `offset_inout`, and vice versa.

```
fun offset_sink2(_ v: sink Vector2, by delta: Vector2) -> Vector2 {
  offset_inout(&v, by: delta)
  return v
}

fun offset_inout2(_ v: inout Vector2, by delta: Vector2) {
  v = offset_sink(v, by: delta)
}
```

### `set` parameters

The `set` convention lets a callee initialize an uninitialized value.
An argument to a `set` parameter is unique, mutable and guaranteed uninitialized at the function's entry.

```
fun init_vector(_ target: set Vector2, x: sink Double, y: sink Double) {
  target = Vector2(x: x, y: y)
}

public fun main() {
  var v: Vector2
  init_vector(&v, x: 1.5, y: 2.5)
  print(v)                         // (x: 1.5, y: 2.5)
}
```

## Method bundles

When multiple methods have the same functionality but differ only in the passing convention of their receiver, they can be grouped into a single *bundle*.

```
extension Vector2 {
  fun offset(by delta: Vector2) -> Vector2 {
    let {
      Vector2(x: x + delta.x, y: y + delta.y)
    }
    inout {
      &x += delta.x
      &y += delta.y
    }
    sink {
      &x += delta.x
      &y += delta.y
      return self
    }
  }
}

public fun main() {
  let unit_x = Vector2(x: 1.0, y: 0.0)
  var v1 = Vector2(x: 1.5, y: 2.5)
  &v1.offset(by: unit_x)           // 1
  print(v1.offset(by: unit_x))     // 2
  let v2 = v1.offset(by: unit_x)   // 3
  print(v2)
}
```

At the call site, the compiler determines the variant to apply depending on the context of the call.
In this example, the first call applies the `inout` variant as the receiver has been marked for mutation.
The second call applies the `sink` variant as the receiver is no longer used aftertward.

Thanks to the link between the `sink` and `inout` conventions, the compiler is able to synthesize one implementation from the other.
Further, the compiler can also synthesize a `sink` variant from a `let` one.

This feature can be used to avoid code duplication in cases where custom implementations of the different variants do not offer any performance benefit, or where performance is not a concern.
For example, in the case of `Vector2.offset(by:)`, it is sufficient to write the following declaration and let the compiler synthesize the missing variants.

```
extension Vector2 {
  fun offset(by delta: Vector2) -> Vector2 {
    let { Vector2(x: x + delta.x, y: y + delta.y) }
  }
}
```

## Projections

Subscripts and computed properties project values rather than returning one.
Unlike in Swift, subscript can be named and are not necessarily bound to a single recevier.
Otherwise, they operate similarly to Swift's subscripts

```
subscript min<T: Comparable>(_ x: T, _ y: T): T {
  if y < x { yield y } else { yield x }
}

fun main() {
  let one = 1
  let two = 2
  print(min[one, two]) // 1
}
```

Note that, because `min(_:_:)` does not return a value, its parameters need not to be passed with the `sink` convention.
Indeed, they do not *escape* from the subscript.

A projection can be assigned to a local binding and used over multiple statements.
For example:

```
fun main() {
  var numbers = Array([3, 1, 6, 5, 4, 6, 2, 0])
  inout slice = numbers[in: 1 ..< 5]
  &numbers.sort()
  print(numbers) // [3, 1, 4, 5, 6, 2, 0]
}
```

Just like methods, subscripts and properties can bundle multiple implementations to represent different variant of the same functionality.

```
type Angle {
  var radians: Double
  
  property degrees: Double {
    let {
      radians * 180.0 / Double.pi
    }
    inout {
      var d = radians * 180.0 / Double.pi
      yield &d
      radians = d * Double.pi / 180.0
    }
    set(new_value) {
      radians = new_value * Double.pi / 180.0
    }
    sink {
      radians * 180.0 / Double.pi
    }
  }
}
```

There are obvious correspondances between Val's and Swift's accessors:

* A `let` subscript is like Swift's `_read` accessor. It is the default in Val, as it allows to project values without transferring their ownership.
* An `inout` subscript is like Swift's `_modify` accessor.
* A `set` subscript is like Swift's `set` accessor.

`sink` subscripts are a bit different.
They can be used to "destructure" a value and extract some of its parts.
Although they overlap with `sink` methods, they serve to optimize cases where the value from which a projection is created is no longer used.

```
fun main() {
  var numbers = Array([3, 1, 6, 5, 4, 6, 2, 0])
  var slice = numbers[in: 1 ..< 5] // last use of 'number'
  _ = slice.pop_first()
  print(slice)
}
```
