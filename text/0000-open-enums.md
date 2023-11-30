- Feature Name: (`open-enums`)<!-- markdownlint-disable MD041 -->
- Start Date: 2022-10-13
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

Add an `#[repr(open)]` attribute for `#[repr(Int)]` enums which disables niche
optimization, making it representationally _open_ and sound to `as` cast and
transmute from any `Int`. It extends `#[non_exhaustive]` to require a wildcard
branch in all contexts.


[ub]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

# Motivation

Enums in Rust have a _closed_ representation, meaning the only valid
representations of that type have discriminants of the variants listed, with any
violation of this being [Undefined Behavior][ub]. This is the right default for
Rust, since it enables niche optimization and ensures values have a known state,
limiting unnecessary and dangerous code paths.

However, a closed enum is not always the best choice for the mindful systems
programmer. There are many use cases in which an open enum is more suitable,
where it's essentially an integer with certain known values.

## Protocol evolution and forwards compatibility

A microservice that intends to be compatible with multiple versions of its own
protocol is best written to be as backwards and forwards compatible with its own
protocol as possible. A user might reach for a `#[non_exhaustive]` enum, but
that only applies at the source level, and so a service can't transparently
carry a value not known at compile time.

An RPC setup like protobuf will prefer using an [open enum][proto-enum], and
so the most natural way to represent that in Rust is as an integer newtype
with associated newtypes.

## Dynamic FFI

Using an `enum` with C FFI is [incredibly dangerous][non-exhaustive-ub],
especially in a dynamically linked library. This is because C/C++ `enum` and
`enum class` are open enums, and so it's sound for them to create an unknown
value and pass it. An open enum can be used much more safely, since it is
allowed to represent these unknown values and must always be checked for in a
`match`. A `#[repr(open, C)]`, a enum can be ergonomically and soundly used
through FFI. The alternatives are `const`s in a module or a newtype integer with
associated constants.

[repr-c-fieldless]: https://doc.rust-lang.org/reference/type-layout.html#reprc-field-less-enums
[non-exhaustive-ub]: https://github.com/rust-lang/rust-bindgen/issues/1763

## Zero-copy (de)serialization

When performing syscalls or reading/writing large data structures in an embedded
system, it's common to have some fields with a small set of named values.
In order to perform this operation as efficiently as possible, a library like
[`zerocopy`][zerocopy-frombytes-derive] can be used to prove a type can be
constructed from any bit pattern, making a `&[u8]` to/from `&T` cast cost solely
a size (and possibly alignment) check.

In these size-constrained systems, it can be more efficient to defer checking
the range of enum values for validity. Much of the code that needs to do these
checks on specific fields in a larger struct can be optimized out when unused in
the binary, while this is much less likely with an
[upfront check][bytemuck-checkedbitpattern] in the validity of a whole struct
with many fields.

[bytemuck-checkedbitpattern]:
    https://docs.rs/bytemuck/latest/bytemuck/checked/trait.CheckedBitPattern.html
[zerocopy-frombytes-derive]:
    https://docs.rs/zerocopy/0.6.1/zerocopy/derive.FromBytes.html

# Guide-level explanation

Enums discriminants have a _closed_ representation by default, meaning that the
enum must be one of their listed variants. This closed representation means that
`Option<Fruit>` can represent `None` with `2u32`, an invalid `Fruit` value,
instead of storing the presence next to the data. This is similar to how
`Option<&T>` is the same size as `&T` through the [null-pointer
optimization][npo]. So, any enum existing with an invalid value is [Undefined
Behavior][ub]. For example, the following is UB:

```rust
#[repr(u32)]     // Fruit is closed, represented with specific values of `u32`
enum Fruit {
    Apple,       // Apple is represented with 0u32
    Orange,      // Orange is represented with 1u32
    Banana = 4,  // Banana is represented with 4u32
}
// Undefined Behavior: 5 is not a valid discriminant for `Fruit`!
let fruit: Fruit = unsafe { core::mem::transmute(5u32) };
```

[npo]: https://doc.rust-lang.org/std/option/#representation

However, by writing `#[repr(u32, open)]` (and `#[non_exhaustive]`), this
indicates that the enum can represent variants that may be added in the future.
`Fruit` can be represented by any `u32`, and in the defining crate, be `as` cast
_from_ `u32`.

```rust
#[repr(u32, open)]  // Fruit is open, represented with any `u32`
#[non_exhaustive]   // All Fruit users must include a wildcard branch in `match`
enum Fruit {
    Apple,         // Apple is represented with 0u32
    Orange,        // Orange is represented with 1u32
    Banana = 4,    // Banana is represented with 4u32
}

let fruit = 5u32 as Fruit;
assert!(!matches!(fruit, Fruit::Apple | Fruit::Orange | Fruit::Banana));
// `fruit` preserves its value casting back to `u32`
assert_eq!(fruit as u32, 5);

let other_fruit = 10 as Fruit;
// `Fruit::Unknown` matches against multiple integer values
assert!(!matches!(fruit, Fruit::Apple | Fruit::Orange | Fruit::Banana));
assert_eq!(other_fruit as u32, 10);

// error: incompatible cast: `Fruit` must be cast from a `u32`
// help: to convert from `isize`, perform a conversion to `u32` first:
//         let fruit: Fruit = u32::try_from(5isize).unwrap() as Fruit;
let third_fruit: Fruit = 5isize as Fruit;

// This is also sound and can be written outside of the defining crate.
let _: Fruit = unsafe { mem::transmute(20u32) };
```

When `#[derive(PartialEq)]` and `#[derive(PartialCmp)]` are applied to a
fieldless open enum, they compare the integer values of the discriminant. This
means that different values of an enum can compare not equal when they are both
unknown discriminant values. Because of this, users should be cautious when
comparing enum values that have not been checked to be a known value.

```rust
#[derive(PartialEq, PartialCmp)]
#[repr(u32, open)]
#[non_exhaustive]
enum Fruit {
    Apple,
    Orange,
    Banana = 4,
}

let f1 = 5u32 as Fruit;
let f2 = 10u32 as Fruit;
assert!(f1 < f2);
```

## Interaction with `#[non_exhaustive]`

`#[repr(open)]` enums almost always require adding `#[non_exhaustive]`, and this
expands the non-exhaustiveness to in-crate users.

Normally, a `#[non_exhaustive]` enum does not require in-crate users of that
enum to include a wildcard branch; it's as if the attribute wasn't there. The
enum is non-exhaustive at a _source code level_ since it solely exists for API
forwards compatibility. However, when `#[repr(open)]` is added, the enum's now
non-exhaustive at a _binary level_ and so all users must include a wildcard
branch, even inside of the defining crate.

```rust
#[derive(PartialEq, PartialCmp)]
#[repr(u32, open)]
#[non_exhaustive]
enum Fruit {
    Apple,
    Orange,
    Banana = 4,
}

match fruit {
  Fruit::Orange | Fruit::Banana => { peel(fruit); eat(fruit) },
  Fruit::Apple => eat(fruit), 
  // wildcard branch always required, even for in-crate users
  unknown => panic!("I don't know how to eat this safely"),
}
```

## Enums with Fields

An enum with fields can be made open as well, so long as it uses one of the
stable representations: `#[repr(Int)]` and/or `#[repr(C)]`, described in [RFC
2195][rfc-2195].

[rfc-2195]: 2195-really-tagged-unions.md#guide-level-explanation

A fielded open enum assumes that unlisted variants can hold arbitrary data too,
which it doesn't know how to interpret. This makes it closer to a `union` than
closed enums are. Because of this, they also come a similar restriction: values
in fielded open enums cannot have drop glue.

```rust
#[repr(u32, open)]
enum Shape {
    Square { width: f32 },
    Circle { radius: f32 },
    // error[E0740]: open enums cannot contain fields that may need dropping
    //  = note: this enum is open because it specifies #[repr(open)]
    //      #[repr(u32, open)],
    //                  ^^^^
    //  = note: a type is guaranteed not to need dropping when it implements
    // `Copy`, or when it is the special `ManuallyDrop<_>` type
    // help: when the type does not implement `Copy`, wrap it inside a
    // `ManuallyDrop<_>` and ensure it is manually dropped
    //      Path { points: std::mem::ManuallyDrop<Vec<(f32, f32)>> },
    Path { points: Vec<(f32, f32)> },
}
```

Open enums with fields are most useful in scenarios where the validity of the
value is not as important as its value remaining untouched, as well as binary
forwards compatibility with future versions of an enum.

```rust
#[repr(u32, open)]
#[derive(Debug)]
enum Shape {
    Square { width: f32 },
    Circle { radius: f32 },
    Ellipse { semi_major: f32, semi_minor: f32 },
}

match shape {
    Shape::Square { width } => self.draw_square(width),
    Shape::Circle { radius } => self.draw_circle(radius),
    Shape::Ellipse { semi_major: a, semi_minor: b } => self.draw_ellipse(a, b),

    // If `shape` has a discriminant of 10, this prints: `unknown shape 10`.
    unknown => warn!("unknown shape {unknown:?}"),
}
```

# Reference-level explanation

When `#[repr(open)]` is applied to an enum:

- It must have an explicit representation (`repr(C)` or `repr(Int)`).
- The discriminant portion of the enum has no invalid values - all values
  of the representation integer are allowed.
- `#[non_exhaustive]` must be present on the enum, _unless_ it has variants for
  all values of `Int` (where `Int` is the underlying integer type).
  - In this exception case, `#[repr(open)]` has no effect on the enum -
    `#[non_exhaustive]` is still allowed but it will apply to in-crate users
    only.
- When pattern matching, matching on a variant does not contribute towards the
  exhaustiveness of the arms. This applies inside and outside of the defining
  crate.
- If the enum has fields:
  - No fields may contain drop glue.
  - It cannot derive `Eq`, `Hash`, `Ord`, `Copy`, or `Clone`.
  - It is an ABI breaking change to add a field with a stricter alignment
    requirement than every other field, since this affects the alignment of the
    whole enum.


## Fielded enum layout

### Union-of-structs representation

A `#[repr(Int, open)]` fielded `enum` now means: the enum must be
represented as a C-union of C-structs that each start with an integer newtype
that is `#[repr(transparent)]` over `Int`. This union also includes an
additional C-struct field used to represent all unrecognized variants.

The other fields of the structs are the payloads of the variants, used to store
the data. Here's an example - this definition:

```rust
#[repr(u32, open)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
}
```

Has the same layout as the following:

```rust
#[repr(C)]
union MyEnumRepr {
    A: MyEnumVariantA,
    B: MyEnumVariantB,
    C: MyEnumVariantC,
    D: MyEnumVariantD,
    Unknown: MyEnumUnknownVariant,
}

#[repr(u32, open)]
enum MyEnumTag { A, B, C, D };

#[repr(C)]
struct MyEnumVariantA(MyEnumTag, u32);

#[repr(C)]
struct MyEnumVariantB(MyEnumTag, f32, u64);

#[repr(C)]
struct MyEnumVariantC { tag: MyEnumTag, x: u32, y: u8 }

#[repr(C)]
struct MyEnumVariantD(MyEnumTag);

#[repr(C)]
struct MyEnumUnknownVariant(MyEnumTag);
```

### `(tag, union)` structure

The more convenient but less efficient `(tag, union)` structure can also be
written with `#[repr(open)]`. When applied to a `#[repr(Int)]` enum, it is laid
out as a C-struct with two fields: a `#[repr(Int)]` open enum containing each of
the tag, and a C-union with a field for every variant's data, including an extra
empty field for unrecognized variants.

So, this enum:

```rust
#[repr(C, u32, open)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
}
```

has the same layout as:

```rust
#[repr(C)]
struct MyEnumRepr {
    tag: MyEnumTag,
    payload: MyEnumPayload,
}

#[repr(u32, open)]
enum MyEnumTag { A, B, C, D };

#[repr(C)]
union MyEnumPayload {
   A: u32,
   B: MyEnumPayloadB,
   C: MyEnumPayloadC,
   D: [u8; 0],
   _Unknown: [u8; 0],
}

#[repr(C)]
struct MyEnumPayloadB(f32, u64);

#[repr(C)]
struct MyEnumPayloadC { x: u32, y: u8 }
```

## Interaction with the standard library

- `derive(Default)` remains unchanged: it still requires a unit variant with a
  `#[default]` annotation.
- `derive(Debug)` uses the `Debug` representation for the integer value for
  unknown variants, though this is allowed to change.
- Fieldless open enums can derive `Clone`, `Copy`, `Eq`, `Hash`, `Ord`,
  `PartialEq`, and `PartialOrd`, which defer to the implementation for the
  representation integer, including for unknown discriminants.
- Fielded open enums can derive `PartialEq` and `PartialOrd`, which act
  similarly to closed fielded enums: first the discriminant value is compared,
  then the data. If two enums have equal discriminants, `PartialEq::eq`
  conservatively returns `false` and `PartialOrd::partial_cmp` returns `None`,
  since the enum has no way to compare the data for equality.
- Fielded open enums cannot derive `Eq`, `Hash`, `Ord`, `Copy`, or `Clone`,
  since there is no reasonable default to interpret unknown variants.
- The [`mem::Discriminant`] of an open enum is equivalent to its `Int`
  representation.

[`mem::Discriminant`]: https://doc.rust-lang.org/core/mem/struct.Discriminant.html

## Integer-exhaustive enums

The open branch requirement specified throughout this RFC only applies to open
enums that can represent unlisted variants. An enum that has a variant for every
value of its underlying integer does not require an open branch, as it is
exhaustive. In this case, specifying the `transparent` makes no difference in
representation or behavior.

# Drawbacks

- This adds extra edge cases to `#[non_exhaustive]`, removing its consistent
  and simple "has no effect inside of the defining crate".
- Every new feature to Rust is another thing to maintain and for users to learn.

# Rationale and alternatives

## Why a language extension? Why not just use an integer newtype or macro?

The best way to write a fieldless open enum in Rust today is the "newtype enum"
pattern that uses associated constants for variants. So, to make this enum open:

```rust
enum Color {
    Red,
    Blue,
    Black,
}
```

the author can write this:

```rust
#[repr(transparent)]  // Optional, but often useful
#[derive(PartialEq, Eq)]  // In order to work in a `match`
struct Color(pub i8);  // Alternatively, make the inner private and `impl From`

#[allow(non_upper_case_globals)]  // Enum variants are CamelCase
impl Color {
    pub const Red: Color = Color(0);
    pub const Blue: Color = Color(1);
    pub const Black: Color = Color(2);
}
```

With this syntax, users of an open enum can use these variant names inside a
`match` with _mostly_ the same syntax as they would with a regular closed enum,
except there must _always_ be an open branch for handling unknown values.
This syntax also provides grouping of related values and associated methods, an
advantage over module-level `const` items.

However, this pattern has some distinct disadvantages:

- This pattern is arduous to read, write, and discover compared to the `enum`
  syntax.
- `enum`s are what Rust users naturally think of for enumerated values.
- Rust is a systems language that can move data around efficiently, and so
  first-class support for named integers valuable for embedded programmers.
- Users of the open enum lose compiler guidance and lints specific to enums.
  - No "fill match arms" in rust-analyzer.
  - The [`non-exhaustive patterns` error][E0004] lists only integer values,
    and cannot suggest the named variants of the enum.
  - The unstable `non_exhaustive_omitted_patterns` lint would have no way to
    work with open enums, which are similarly non-exhaustive.
- This pattern hides the type's nature as an enum.

The newtype enum pattern only works for fieldless enums. This RFC also enables
open enums that carry data, which otherwise require manual use of `union` to
implement correctly while losing safety for known variants and `match`
capability.

[E0004]: https://doc.rust-lang.org/stable/error-index.html#E0004

## Fieldless open enums _are_ tuple structs

Rather than using `as` casts to convert an integer to an enum, an open enum acts
as a tuple struct while still using the `enum` syntax to declare. This would
allow for more ambitious `match`es and extracting the integer value in the same
way as a newtype enum. It would also allow people who are using the newtype enum
pattern now to replace with an open enum with no breakage, a major win for
language evolution.

```rust
match color {
    Color::Red => println!("red"),
    Color::Blue => println!("blue"),
    Color::Green => println!("green"),
    Color(x @ ..200) => println!("known-valid color '{x}'"),
    // The current proposed syntax would use an `if` guard instead:
    // x if matches!(x as u32, ..200) => println!("in-spec color '{x}'"),
    Color(x) => println!("color '{x}' outside of known-valid range"),
}
// Since it's a tuple struct, we can also access the inner value and even
// mutate it with `.0`.

println!("{}", color.0);
color.0 += 1;
```

However, this:

- Introduces a syntax that is not obvious from the declaration.
- Does not mirror the infallible conversion of `as` conversion to integers.
- Seeing `.0` on an `enum` is unexpected.
- Always exposing the integer value of an enum provides more access to the
  internals of the enum than may be desired.

## The other variant carries its unknown values like a tuple variant

An alternative way to specify a fieldless open enum could be to write this:

```rust
#[repr(u32)]
enum IpProto {
   Tcp = 6,
    Udp = 17,

    // bikeshed syntax
    Other(..),
}
```

This would mean that the `Other` variant is a named way to refer to unlisted
values and works in pattern matching naturally, while being a zero-cost
representation:

```rust
if let IpProto::Other(x) = proto {
    // `proto` was *not* `Tcp` or `Udp`; its integer value is in `x`.
}
```

It also reduces the `#[derive(PartialEq)]` and `#[derive(PartialCmp)]` footgun
of the original design, since constructing the rest variant requires inputting

However, this has some problems. For one, it's peculiar for a tuple variant
syntax to leave a fieldless enum fieldless. Also, this could lead to surprising
behavior with pattern matching, unless the match semantics are changed greatly:

```rust
if let IpProto::Other(x) = IpProto::Other(6) {
    // This branch is not taken, since it's actually an `IpProto::Tcp`!
}
```

This _could_ do a check that the provided integer value isn't a known variant,
but that generates an implicit panic for what looks like a simple enum
construction. Instead, to get this behavior with this RFC's proposed syntax, the
author can use a derive to check against the listed variants, of which many
exist already:

```rust
#[repr(transparent, u32)]
#[derive(IsKnownVariant)]
enum IpProto {
    Tcp = 6,
    Udp = 17,
}

if !proto.is_known_variant() {
    let x = proto as u32;
    // `proto` was *not* `Tcp` or `Udp`; its integer value is in `x`.
}
```

# Prior art

_Open_ and _closed_ enums are [pre-existing industry terms][acord-xml].

## Enum openness in other languages

- C++'s [scoped enumerations][cpp-scoped-enums] are open enums.
- C# uses [open enums][cs-open-enums], with a [proposal][cs-closed-enums] to add
  closed enums for guaranteed exhaustiveness.
- Java uses closed enums.
- [Protobuf][protobuf-enum] uses closed enums with the `proto2` syntax, treating
  unlisted enum values as unknown fields, and changed the semantics to open
  enums with the `proto3` syntax. This was in part because of lessons learned
  from protocol evolution and service deployment as described above.

[acord-xml]: https://docs.oracle.com/cd/B40099_02/books/ConnACORDFINS/ConnACORDFINSApp_DataType10.html
[cpp-scoped-enums]: https://en.cppreference.com/w/cpp/language/enum#Scoped_enumerations
[cs-open-enums]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/enum#conversions
[cs-closed-enums]: https://github.com/dotnet/csharplang/issues/3179
[protobuf-enum]: https://developers.google.com/protocol-buffers/docs/reference/cpp-generated#enum

## Other crates that use the "newtype enum" pattern

- [open-enum][open-enum], written by the author of this RFC. It's a procedural
  macro which converts any fieldless `enum` definition to an equivalent newtype
  struct with associated constants.
- Bindgen is [aware of the problem][bindgen-ub] with FFI and closed enums, and
  avoids creating Rust enums from C/C++ enums because of this. It provides an
  option for newtype enums directly.
- `winapi-rs` defines an [`ENUM`][winapi-enum] macro which also generates this
  pattern for simple `enum` definitions.
- ICU4X uses newtype enums for [certain properties][icu4x-props] which must be
  forwards compatible with future versions of the enum.
- [OpenTitan][opentitanlib-unknown]

The [`newtype-enum` crate][newtype-enum-crate] is an entirely different pattern
than what is described here.

[bindgen-ub]: https://github.com/rust-lang/rust/issues/36927
[icu4x-props]: https://github.com/unicode-org/icu4x/blob/ff1d4b370b834281e3524118fb41883341a7e2bd/components/properties/src/props.rs#L56-L106
[newtype-enum-crate]: https://crates.io/crates/newtype-enum
[open-enum]: https://crates.io/crates/open-enum
[opentitanlib-unknown]: https://github.com/lowRISC/opentitan/blob/master/sw/host/opentitanlib/src/util/unknown.rs
[winapi-enum]: https://github.com/retep998/winapi-rs/blob/77426a9776f4328d2842175038327c586d5111fd/src/macros.rs#L358-L380

# Unresolved questions

## Is it right to add this edge case to `#[non_exhaustive]` behavior?

This RFC goes "one step" farther than `#[non_exhaustive]`: it's
non-exhaustive at a binary level, and so also requires same-crate users to treat
it as non-exhaustive. There could be a modification to the syntax, like
`#[non_exhaustive = "always"]`, to highlight that defining-crate users must
still perform non-exhaustive matches on the enum.

## `Int as Enum` casting 

Currently, the design allows `as` casting from the explicit underlying integer
type on an open enum to the enum type inside of the crate only. This is
because it is currently disallowed for a `#[non_exhaustive]` enum to be
`as` cast _to_ an integer while outside of the defining crate.

This RFC could instead forbid `as` casting to the enum altogether, or allow it
in all cases.

# Future possibilities

## Extracting the integer value of the discriminant for fielded enums

A fielded enum with `#[repr(Int)]` and/or `#[repr(C)]` is guaranteed to have its
discriminant values starting from 0. However, for any given value of that enum,
there's no built-in way to extract what the integer value of the discriminant is
safely. The unsafe mechanism is `(&thenum as *const _ as *const Int).read()`.
For open fielded enums, this would be even more valuable, since the discriminant
could be entirely unknown and the programmer may want to know its value. Perhaps
an extension to [`mem::Discriminant`]?

## Variants with discriminant ranges

Rust could allow variants that correspond to range of discriminants instead of a
single discriminant. This would allow for:
- a natural way to create an `Unknown` variant to `match` against rather than
  requiring a wildcard branch to check for this case as `#[repr(open)]` does.
- Reserving ranges of enum values for ABI compatibility
- Defining a C boolean as an enum

```rust
/// Compatible with a C/C++ `bool`.
/// This is also an open enum since there are no niches.
#[repr(u8)]
enum CBool {
    False = 0,
    True = 1..=c_int::MAX,
}

/// An error code enum that is ABI-compatible with values up to 1023.
#[repr(C)]
enum ErrorCode {
    NotFound = 1,
    Forbidden = 2,
    AlreadyThere = 3,
    Reserved = 4..=1023,
}
```
