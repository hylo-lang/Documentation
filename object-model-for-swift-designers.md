
# The Val Object Model

[Val](https://www.val-lang.dev) is a Swift-inspired language being developed by @dabrahams, @Alvae,
and a handful of others.

Beyond trivial syntactic similarity, Val draws on Swift for its object model, re-imagining how we
think it would look if Swift:
- Started with support for non-copyable types.
- Had the benefit of insights gained during its development, especially the law of exclusivity and
  `_modify`/`_read` accessors.
- Did not start with support for (single-thread safe) reference semantics at its core.

This post is not an attempt to sell Val to anyone, and we certainly don't believe Swift can adopt
Val's object model wholesale, immediately, or without modification.  That said, we think Swift might
be evolved in the direction of Val's model, and that doing so could result in a simpler, better
Swift.  Because Val was designed with noncopyable types in mind, the model is cleaner than what's
currently being contemplated for the future of Swift, which understandably shows signs of a
retrofit.  At @Joe_Groff's
[suggestion](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581/62) we
are starting this thread to introduce the big picture in one place, tuned to be appropriate for the
Swift evolution audience.

Significant portions of this post synthesize parts of [Val's language
tour](https://www.val-lang.dev/pages/language-tour.html), with additional comments describing how
Val relates to Swift.  We have omitted detailed explanations of things we think this audience can
easily intuit on their own, but please feel free to ask questions about anything.

## Pure Value Semantics

Val's originating question was: *What does programming look like when there is only [mutable value
semantics](https://www.jot.fm/issues/issue_2022_02/article2.pdf)?*

Therefore we started with Swift, omitting the three sources of reference semantics in Swift's
(single-thread) safe subset:
- classes
- mutable global variables
- closure captures with reference semantics

The approach is a major departure from that of Swift, but we believe the design is relevant
nonetheless, and that reference semantics in the style of Swift can be reconstructed on top of Val's
core object model. [Note that shared mutable state is accessible in the Val model's unsafe subset
via pointer dereferences].

## Explicit, necessary copies

The way Swift passes arguments to functions showed us that arguments passed by value don't need to
be copied or moved, which led us to realize that the semantics of a `let` binding (which can be
viewed as passing by-value to a continuation function) doesn't necessarily imply a copy or move
either.

When these copies are eliminated, the need for copying becomes rare enough that:

- It becomes reasonable to make all copies, even those of trivial types like `Int`,
  explicit via `x.copy()`, thus simplifying the story for generic code that must operate on both
  kinds of type.
- Many generic algorithms written straightforwardly to operate on copyable types “just work” on
  non-copyable types without modification, so the need to add a generic `Copyable` requirement
  becomes rare.
- Since a copy is only needed when the source of a `let` binding is modified during the `let`
  binding's lifetime, it turns out that the compiler can issue a “fix-it” suggesting exactly the
  necessary copies, and can warn about those that are unnecessary.  The user experience is analogous
  to the one Swift provides around `let` and `var.
- Finally, since the compiler can identify necessary and unnecessary copies, we can add a way to
  turn on implicit copies.
  
In the latter mode **the programming model for copyable types is practically identical to the one
that Swift offers today**, but with only the necessary copies actually being made. It is this
overlap that makes us think it may be possible to evolve Swift toward Val's model.  We also think
this model provides a suitable platform for interoperating with C++.  Swift may need to use the
opposite default for backward compatibility reasons of course.

### Examples

#### Missing copy detection:

The ability to create a let binding without an implicit copy means that immutability must be
conferred upon the source of the binding during the binding's lifetime.

```
fun f(x: inout Int) {
  let y = x // <-------------------------------------------+
  x += 1    // error: x is let-bound.  Insert a copy here -+
  print(x, y)
}
```

If Swift ever gets local `inout` bindings, an analogous coupling will confer inaccessiblity on
source values during the lifetime of the binding.

Because by-value parameters are borrowed, they can't be stored or returned without a copy:

```
type Vector2 {
  var x: Double
  var y: Double

  /// Creates an instance whose `x` and `y` values are `buffer[0]` and 
  /// `buffer[1]` respectively.
  init(_ buffer: Double[2]) {
    x = buffer[0].copy()
    y = buffer[1] // error: `buffer[1]` cannot escape; insert a copy here.
  }
}
```

An alternative would be to pass `buffer` as a `sink` parameter, which would allow it to be consumed
to create the new `Vector2` instance.  More on passing conventions later.

#### Opt-in implicit copying

Because the compiler can know where copies would have been made in a language with implicit
copying, we can allow the user to enable implicit copies, resulting in a model that mimics Swift.

```
@implicitcopy

type Vector2 {
  var x: Double
  var y: Double

  init(_ buffer: Double[2]) {
    x = buffer[0] // implicit copy here
    y = buffer[1] // and here
  }
}
```

*Note: you can use `@implicitcopy` at any scope, or can parameterize it with specific types or
traits (e.g. `CheaplyCopyable`), allowing the effect to be controlled when it matters.*

#### Copyability

Copyability is enable by making types conform to the `Copyable` trait.  A trait is like Swift protocol.

```
type Vector2: Copyable {
  var x: Double
  var y: Double
}
```

Just as in Swift, conformance to traits like `Copyable`, `Equatable` and `Hashable` can be
synthesized when stored properties have those conformances.  

## Parameter passing conventions

A parameter passing convention describes how an argument is passed from caller to callee.
In Val, there are four: `let`, `inout`, `sink` and `set`.
The next series of examples define four corresponding functions to offset this 2-dimensional vector type:

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

The `let` convention does not transfer ownership of the argument to the callee, meaning, for
example, that without first copying it, a `let` parameter can't be returned, or stored anywhere that
outlives the call.

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

One syntactic difference from Swift you'll notice is that in Val there are no exceptions to
the rule that mutated objects are marked with `&`, so you'll see it in uses of operators and
mutating methods.

Another difference from Swift is that, although `inout` parameters are required to be valid at
function entry and exit, a callee is entitled to do anything with the value of such parameters,
including destroying them, as long as it puts a value back before returning.  That makes it
convenient to temporarily move an argument into an object that encapsulates some computation.

```
type Processor {
  var ast: AST
  fun process() inout { ... }
  fun process_expr() inout { ... }
  fun process_decl() inout { ... }
  fun finalize() sink -> {AST, Result} { ... }
}

fun process(_ ast: inout AST) -> Result {
  // Move `ast` into `p`, without copying it
  var p = Processor(ast)
  
  // Execute the computation encapsulated by `Processor`
  &p.process()

  // Move `p` back into `ast`.
  let result: Result
  (ast, result) = p.finalize()
  return result
}
```

*Note: In Val, the passing convention for the `self` parameter to a method is written after the
parameter list, so mutating methods are marked with a postfix `inout`*.

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
  
  print(v) // <== error: v was consumed above.
}          // to use v here, pass v.copy() to offset_sink.
```

<a name="let_inout_sink"></a> The conventions we've seen so far are closely related; so much so that
`offset_sink` can be written in terms of `offset_inout`, and vice versa:

```
fun offset_sink2(_ v: sink Vector2, by delta: Vector2) -> Vector2 {
  offset_inout(&v, by: delta)
  return v
}

fun offset_inout2(_ v: inout Vector2, by delta: Vector2) {
  v = offset_sink(v, by: delta)
}
```

Furthermore, either one can be written in terms of `offset_let`:

```
fun offset_sink3(_ v: sink Vector2, by delta: Vector2) -> Vector2 {
  offset_let(v, by: delta)
}

fun offset_inout3(_ v: inout Vector2, by delta: Vector2) {
  v = offset_let(v, by: delta)
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
  
  &v1.offset(by: unit_x)          // inout
  let v2 = v1.offset(by: unit_x)  // let
  print(v1.offset(by: unit_x))    // sink
  print(v2)
}
```

At the call site, the compiler determines the variant to apply depending on the context of the call.
In this example, the first call applies the `inout` variant as the receiver has been marked for mutation.
The second call applies the `let` variant as the receiver is used in the next line
The third call applies the `sink` variant as it is receiver's last use.

Thanks to the [link](#let_inout_sink) between conventions, the compiler is able
to synthesize one implementation from the other as long as the type is `Sinkable`. 
This feature can be used to avoid code duplication in cases where custom implementations of the different variants do not offer any performance benefit, or where performance is not a concern.
For example, in the case of `Vector2.offset(by:)`, it is sufficient to write the following declaration and let the compiler synthesize the missing variants.

```
extension Vector2 {
  fun offset(by delta: Vector2) -> Vector2 {
    let { Vector2(x: x + delta.x, y: y + delta.y) }
  }
}
```

### The `Copyable` and `Sinkable` traits.

The `Copyable` trait is a refinement of `Sinkable`, the trait for types that can be moved.  They are
defined as follows in the standard library, using method bundles:

```
trait Sinkable {

  // Gives `self` the value of a consumed `source`.
  fun take_value(from source: sink Self) {
    // The pattern `let x: T = f()` is compiled as `let x: T; x.take_value(from: f())`.
    set { /* synthesized memberwise */ }   // move-initialization

    // The pattern `x = f()` is compiled as `x.take_value(from: f())`.
    inout { /* synthesized memberwise */ } // move-assignment
  }

}

trait Copyable: Sinkable {

  // Gives `self` the copied value of `source`.
  fun copy_value(from source: sink Self) {
    // The pattern `let x: T = expr.copy()` is compiled as `let x: T; x.copy_value(from: expr)`.
    set { take_value(from: source.copy()) }   // copy-initialization

    // The pattern `x = y.copy()` is compiled as `x.copy_value(from: y)`.
    inout { take_value(from: source.copy()) } // copy-assignment
  }

}
```

The compiler chooses an implementation based on the whether the left-hand-side is initialized and
whether the RHS is consumed.

## Projections

A subscript or computed property *projects* a value rather than returning one.
Unlike in Swift, a subscript in Val can be named and can be freestanding, like a function (as
opposed to a method).
Otherwise, it operates similarly to a Swift subscript.

```
// A freestanding named subscript.
subscript min<T: Comparable>(_ x: T, _ y: T): T {
  if y < x { yield y } else { yield x }
}

fun main() {
  let one = 1
  let two = 2
  print(min[one, two]) // 1
}
```

Note that, because `min(_:_:)` `yield`s rather than `return`ing a value, its parameters do not
escape from the subscript.

A projection can be assigned to a local binding and used over multiple statements.
For example:

```
fun main() {
  var numbers = Array([3, 1, 6, 5, 4, 6, 2, 0])
  inout slice = numbers[in: 1 ..< 5] // slice is projected out of `numbers`
  &slice.sort()
  print(numbers) // [3, 1, 4, 5, 6, 2, 0]
}
```

Just like methods, subscripts and properties can bundle multiple implementations to represent different variants of the same functionality.

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

Most of Val's accessors correspond to Swift's:

* A `let` subscript is like Swift's `_read` accessor. It is the default in Val, as it allows to project values without transferring their ownership.
* An `inout` subscript is like Swift's `_modify` accessor.
* A `set` subscript is like Swift's `set` accessor.

However, `sink` subscripts are a bit different.
They can be used to "destructure" a value and extract some of its parts.
Although they overlap with `sink` methods, they serve to optimize cases where the value from which a projection is created is no longer used.

```
fun main() {
  var numbers = Array([3, 1, 6, 5, 4, 6, 2, 0])
  var slice = numbers[in: 1 ..< 5] // last use of 'numbers'
  _ = slice.pop_first()
  print(slice)
}
```
