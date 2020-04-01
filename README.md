# Value Types (i.e., 'struct') for ECMAScript

> TODO: Replace this with a summary or introduction for your proposal.

## Status

**Stage:** 0  
**Champion:** Ron Buckton (@rbuckton)  

_For detailed status of this proposal see [TODO](#todo), below._  
<!--#endregion:status-->

<!--#region:authors-->
## Authors

* Ron Buckton (@rbuckton)  
<!--#endregion:authors-->

<!--#region:motivations-->
# Motivations

> TODO: Replace this with motivations and use cases for the proposal.
<!--#endregion:motivations-->

# Prior Art 

> TODO: Add links to similar concepts in existing languages, prior proposals, etc.

* Language: [Concept](url)  

# Syntax

```js
struct Point {
  x: i32;
  y: i32;

  offset(dx, dy) { return Point(this.x + dx, this.y + dy); }
  toString() { return `${this.x},${this.y}`; }

  static (Point + Point)  (a, b) { return Point(a.x + b.x, a.y + b.y); }
  static (Point + Number) (a, b) { return Point(a.x + b, a.y + b); }
  static (Point - Point)  (a, b) { return Point(a.x - b.x, a.y - b.y); }
  static (Point - Number) (a, b) { return Point(a.x - b, a.y - b); }
  static (Point * Number) (a, b) { return Point(a.x * b, a.y * b); }
  static (Point / Number) (a, b) { return Point(a.x / b, a.y / b); }

  static (- Point) (a) { return Point(-a.x, -b.x); }
  static (~ Point) (a) { return Point(~a.x, ~b.x); }
}
```

# Semantics

## Structs

A *struct* is a declaration for a data structure with *value semantics*. This means:
- A *struct value* consists of a two pieces of information at runtime:
  - A *value* consisting of a contiguous chunk of memory encapsulating all of the data declared by the *struct value*'s *struct* declaration.
  - A *type* which is a reference to the *struct* declaration that was used to allocate the *struct value*.
  - NOTE: This is analogous to the behavior of a value such as `0`, which includes both its *value* (`0`) and its *type* (`Number`).
- A *struct value* is copied on assignment, when passed as the argument to a function, or when passed as the expression to a `return`, `throw`, `yield`, or `await`.
- A *struct value* is either *mutable* (if declared using `struct`), or *immutable* (if declared using `const struct`).
  - A *struct value* that is *immutable* cannot be modified. Attempting to mutate such a value will result in an error in strict-mode contexts.
  - A *struct value* this is *mutable* can be modified. Such modifications change the underlying memory of the struct, but do not affect *copies* of the struct.
- Calling `ToObject` on a *struct value* is similar to what happens for other primitive values, in that a new
  ECMAScript `Object` is allocated whose `[[Prototype]]` slot points to the `prototype` property of the *struct*'s constructor function, and whose `[[StructData]]` internal slot is set to the *struct value*.

**Examples**
```js
struct Mutable {
  // ...
}
const struct Immutable {
  // ...
}
```

## Fields

A *struct* consists of zero or more *fields* which define the structure, size, and organization of the underlying memory allocated for a *struct value*.
As the memory allocated for a *struct value* must be contiguous, the type and size for each field generally must be known ahead of time and as such must be
included as part of the *struct*'s declaration. 

A *field* may be declared using _IdentifierName_, _NumberLiteral_, _StringLiteral_, _ComputedPropertyName_, or _PrivateIdentifier_. Similar to `class` declarations,
a _PrivateIdentifier_ is lexically scoped to the body of the `struct` declaration.

A *field* may have an optional initializer. A *field* without an initializer will be initialized using a 
default value derived from the *field*'s declared type.

**Examples**
```js
struct Point {
  x: i32;
  y: i32;
  // ...
}

const struct Decimal {
  #sign: i8;
  #exponent: i32;
  #coefficientLength: i32;
  #coefficient: i32[#coefficientLength];
  // ...
}
```

### Field Types

A *field*'s *field type* is mandatory and must be specified following the *field*'s name, separated by 
a colon (`:`). A *field type* may consist of one of the following:

- A built-in reserved *primitive type*:
  - `i8` - A one-byte signed integer.
  - `i16` - A two-byte signed integer.
  - `i32` - A four-byte signed integer.
  - `i64` - An eight-byte signed integer.
  - `u8` - A one-byte unsigned integer.
  - `u16` - A two-byte unsigned integer.
  - `u32` - A four-byte unsigned integer.
  - `u64` - An eight-byte unsigned integer.
  - `f32` - A four-byte floating point number.
  - `f64` - An eight-byte floating point number.
  - `box` - A memory-managed reference to an ECMAScript language value whose size is host-dependent.
- An _IdentifierReference_ that evaluates to the constructor from a *struct* declaration (i.e., `Point`, 
  etc.).

In addition, the *field type* may be suffixed by an *array length indicator* using one of the following forms:

- A fixed-size array, such as `i32[8]` (in this case indicating an `Int32Array` of length `8`).
- A dependent-size array, such as `i32[length]` (in this case indicating an `Int32Array` whose length is derived from the *struct value*'s `length` field.).
  - NOTE: In a *mutable* struct, changing `length` would resize the _TypedArray_, possibly causing data loss.
- A flexible-size array, such as `i32[]`, which may only occur as the last field in a *struct*.

## Constructors

A *struct* declaration's `[[Construct]]` slot is overloaded, such that every *struct*'s constructor can always be called with the following arguments:

- `(typedArray: TypedArray)` - Allocates a *struct value* from a `TypedArray`.
- `(buffer: ArrayBuffer | SharedArrayBuffer, byteOffset?: number)` - Allocates a *struct value* from an `ArraBuffer` or `SharedArrayBuffer`.

If neither of these overloads conditions are met, construction of the struct falls to either its declared
`constructor`, or a default constructor. The default constructor for a *struct* consists of a parameter 
for each field, in declaration order. Each parameter can be considered to have an initializer consistent with
the *field*'s initializer.

**Examples**
```js
struct Point {
  x: i32;
  y: i32;
}
const p1 = Point();
console.log(p1.x, p1.y); // 0 0
const p2 = Point(10);
console.log(p2.x, p2.y); // 10 0
const p3 = Point(10, 20);
console.log(p3.x, p3.y); // 10 20
```

## Methods/Accessors

A *struct* may declare methods and accessors which are stored on the `prototype` property of the *struct*'s constructor function. These behave similar to prototype members for `Number`, `Boolean`, etc.

**Examples**

```js
struct Point {
  /* ... */
  offset(dx, dy) { return Point(this.x + dx, this.y + dy); }
  toString() { return `${this.x},${this.y}`; }
  /* ... */
}
```

## Operator Overloading

A *struct* declaration supports syntactic *operator overloading*, allowing you to specify how a limited 
subset of ECMAScript binary and unary operators interact with *struct* values.

An *operator method* is a `static` method of the *struct* that specifies both the operator and the valid
*operator types* for each operand:

- `( Type1 BinaryOperator Type2 )` - Indicates a binary operation between `Type1` (on the left) and `Type2` on the right, i.e. `(Point + Point)`. At least one of the two types must refer to the current *struct*.
- `( UnaryOperator Type?)` - Indicates a unary operation on `Type`, which must refer to the current *struct*. 
 - NOTE: For convenience, the `Type` may be elided.

An *operator type* must be an _IdentifierReference_ pointing to the one of the following:
 - The name of a *struct* declaration.
 - One of a limited set of built-in primitive constructors:
  - `String` - for string values.
  - `Symbol` - for symbol values.
  - `Number` - for number values.
  - `BigInteger` - for bigint values.
  - `Boolean` - for boolean values.
  - `Object` - for everything else.

**Examples**

```js
struct Point {
  // binary operator overloading
  static (Point + Point)  (a, b) { return Point(a.x + b.x, a.y + b.y); }
  static (Point + Number) (a, b) { return Point(a.x + b, a.y + b); }
  static (Point - Point)  (a, b) { return Point(a.x - b.x, a.y - b.y); }
  static (Point - Number) (a, b) { return Point(a.x - b, a.y - b); }
  static (Point * Number) (a, b) { return Point(a.x * b, a.y * b); }
  static (Point / Number) (a, b) { return Point(a.x / b, a.y / b); }
  
  // unary operator overloading
  static (- Point) (a) { return Point(-a.x, -b.x); }
  static (~ Point) (a) { return Point(~a.x, ~b.x); }
}
```

## Future Additions

The following are several possible future additions being considered for this or a follow-on proposal:

- Controlling memory pack behavior using an `@pack` decorator.
- Controlling field offset using an `@fieldOffset` decorator.

# Examples

> TODO: Provide examples of the proposal.

```js
```

# API

Every *struct* constructor has the following members:

- `SIZE` - The minimum number of bytes required to allocate an instance of the *struct*.
- `Array` - A *TypedArray*-like constructor whose elements are based on the *struct*.

Every *struct* declaration has a `prototype` that includes the following members:

- `buffer` - The `ArrayBuffer` backing the *struct value*.
- `byteOffset` - The offset, in bytes, from the start of `buffer` at which this *struct value* starts.
- `with(attrs)` - A method that returns a copy of the *struct value* with its fields values changed based on the own properties of `attrs` (e.g. `Point().with({ x: 10 }) === x`).

# Grammar

> TODO: Provide the grammar for the proposal. Please use [grammarkdown][Grammarkdown] syntax in 
> fenced code blocks as grammarkdown is the grammar format used by ecmarkup.

```grammarkdown
```

# References

> TODO: Provide links to other specifications, etc.

* [Title](url)  

# Prior Discussion

> TODO: Provide links to prior discussion topics on https://esdiscuss.org.

* [Subject](https://esdiscuss.org)  

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [ ] Identified a "[champion][Champion]" who will advance the addition.  
* [ ] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [ ] Illustrative [examples][Examples] of usage.  
* [ ] High-level [API][API].  

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].  
* [ ] [Transpiler support][Transpiler] (_Optional_).  

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].  
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].  
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  

[Process]: https://tc39.github.io/process-document/
[Proposals]: https://github.com/tc39/proposals/
[Grammarkdown]: http://github.com/rbuckton/grammarkdown#readme
[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://rbuckton.github.io/proposal-struct
[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo