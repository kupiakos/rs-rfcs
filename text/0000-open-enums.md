- Feature Name: (`open-enums`)<!-- markdownlint-disable MD041 -->
- Start Date: 2022-10-13
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

Enable first-class support for _open_ enums by specifying a _open variant_ with
the open range `..` as a discriminant. This special variant represents all
unlisted integer values, making every bit pattern a valid discriminant for that
enum. This is in contrast to the default _closed_ representation of Rust enums,
where a discriminant in an enum not listed in its type is [Undefined
Behavior][ub].

[ub]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

# Motivation

Enums in Rust have a _closed_ representation, meaning the only valid
representations of that type are the variants listed, with any violation of this
being [Undefined Behavior][ub]. This is the right default for Rust, since it
enables niche optimization and ensures values have a correct state, limiting
unnecessary and dangerous code paths.

However, a closed enum is not always the best choice for the mindful systems
programmer. There are many specific use cases in which an open enum is more
suitable. These stories are based on real-world examples:

## Protocol evolution and forwards compatibility

Fish is writing a microservice that receives an RPC from one server and forwards
it to another based on the contents of the message, with one of the message's
fields being an integer enum type shared between all parties. Their services
perform swimmingly, until a new feature means adding a new enum variant. They
first roll out the update to final destination servers, then the change begins
rolling out to all other systems with no particular order. Since their structure
cannot represent unknown enums, their service has a partial outage as some
senders are updated before the intermediaries which reject the unknown value.

In order to prevent this problem in the future, Fish chooses to use a different
solution than fieldless `enum`s, even though this works so well with their
tooling and they like the `enum` syntax. This is the better option than
complicating their deployment system further.

## Dynamic FFI

Sebastian is interoperating with a another library that uses a discriminated
union with a plain integer as the discriminant. So, he uses the `#[repr(C,
u32)]` layout for a Rust `enum` with data. Unfortunately, a future version of
the library adds a new variant, and the Rust code that dynamically linked
against the library now invokes UB when it receives that new variant! If he had
written this in C, he would put a `default` case for the `switch` statement on
the discriminant, so that's what he's looking for in Rust. Sebastian then tries
adding `#[non_exhaustive]`. He then later discovers that this is a
*source-level* non-exhaustiveness, not binary, and so the compiler still assumes
unlisted variants are illegal. This means it [does not help prevent his
problem][non-exhaustive-ub]. Disappointed, Sebastian changes his binding to be
fallible, internally using `MaybeUninit<TheEnum>` and a conversion function
which manually extracts and checks the discriminant `unsafe`ly before doing an
`.assume_init()`.

Current `#[repr(C)]` fieldless `enum`s are also [rarely the right
choice][repr-c-fieldless] for representing C enums, and so Sebastian must use a
less ergonomic syntax like `const`s in a module or a newtype integer with
associated constants. Fortunately, `bindgen` does most of this work for him.

[repr-c-fieldless]: https://doc.rust-lang.org/reference/type-layout.html#reprc-field-less-enums
[non-exhaustive-ub]: https://github.com/rust-lang/rust-bindgen/issues/1763

## Embedded programming

Ariel is writing embedded Rust that reads a large data structure from flash,
with some fields having a small set of named values. Coming from C/C++, the team
has written `enum`s in order to express this. However, during implementation,
she realizes that the structure cannot be converted directly from bytes, as her
[zerocopy-deserialization library tells her][zerocopy-frombytes-derive]. Worse,
if Ariel had not realized this, she might have invoked UB unexpectedly.

Ariel likes using `enum`s and would rather not refactor the hundreds they
already have, so she tries to make this sound by performing validation. This
involves an `unsafe` and complex dance in which she uses `addr_of!` to access
possibly-invalid fields to check that they are correct before using.
Fortunately, she discovers another zerocopy-deserialization library can [derive
these checks for her][bytemuck-checkedbitpattern].

Unfortunately, Ariel is also on the hunt for unnecessary code as an embedded
developer. She discovers that the validity checks are shared between multiple
firmware units, and those units all care about different fields. With this
pre-checking and a modular design that separates flash loading from data
manipulation, the optimizer does not recognize which fields actually need to be
verified, resulting in bloated code size that cannot be optimized away. For a
large structure, this adds up fast. With an open enum, she would have only paid
for the necessary branches at the single field-reading site. Realizing this, she
rewrites every `enum` stored in flash as a transparent integer newtype with
associated constants.

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

However, what if we know that `Fruit` can hold unknown values? We can declare an
_open variant_ with `..` that represents the rest of the unrepresented
integer values. This gives the enum an _open representation_, with every bit
pattern being a valid `Fruit`.

```rust
#[repr(u32)]       // Fruit is open, represented with any `u32`
enum Fruit {
    Apple,         // Apple is represented with 0u32
    Orange,        // Orange is represented with 1u32
    Banana = 4,    // Banana is represented with 4u32
    Unknown = ..,  // Unknown is represented with any other `u32`
}
```

An open variant represents multiple integer values, so using it in a pattern
checks that it doesn't equal any of the listed values, rather than checking that
it has a specific integer value:

```rust
match fruit {
    Fruit::Unknown => println!("unknown fruit with value {}", fruit as u32),
    _ => println!("some known fruit"),
}
```

Since the enum is now open and holds any `u32`, it can be cheaply cast with
`as`:

```rust
let fruit = 5u32 as Fruit;
assert!(matches!(fruit, Fruit::Unknown));
// `fruit` preserves its value casting back to `u32`
assert_eq!(fruit as u32, 5);

let other_fruit = 10 as Fruit;
// `Fruit::Unknown` matches against multiple integer values
assert!(matches!(other_fruit, Fruit::Unknown));
assert_eq!(other_fruit as u32, 10);

// error: incompatible cast: `Fruit` must be cast from a `u32`
// help: to convert from `isize`, perform a conversion to `u32` first:
//         let fruit: Fruit = u32::try_from(5isize).unwrap() as Fruit;
let third_fruit: Fruit = 5isize as Fruit;
```

When used in an expression, an open variant has the smallest unlisted
integer value in the enum's type definition lowest unlisted value, with
nonnegative numbers considered first.

```rust
assert_eq!(Fruit::Unknown as i32, 2);

#[repr(i32)]
#[derive(PartialEq, Eq)]
enum Dimension {
    X,
    Y,
    Z,
    Elsewhere = ..,
}

assert_eq!(Dimension::Elsewhere as i32, 3);
```

When `#[derive(PartialEq)]` and `#[derive(PartialCmp)]` are applied to a
fieldless open enum, they compare the integer values of the discriminant. This
means that different values of a given open variant will compare not equal.
Because of this, users should be cautious when comparing values against a
named open variant, since it only checks against a specific unnamed value.

```rust
// Probably incorrect, compares not equal to 3:
assert_ne!(-1i32 as Dimension, Dimension::Elsewhere);

// Probably correct, compares not equal to 0, 1, or 2:
assert!(matches!(-1i32 as Dimension, Dimension::Elsewhere));
```

## Interaction with `#[non_exhaustive]`

Open variants and `#[non_exhaustive]` both involve "unlisted" cases of an enum, and so they interact in a particular way.

`#[non_exhaustive]` applied to an enum solely means that it is not a breaking
change at a source code level to add a new enum variant, so foreign crate users
must include an open branch when matching when compiling against future
versions of an enum to prevent an upgrade causing a compile failure. It does not
affect local crate users.

The effect is best demonstrated with an example:

```rust
// Foreign crate
#[non_exhaustive]
#[repr(u32)]
enum Bone {
    Tibia = 0,
    Fibula = 1,
    Spooky = ..,
}

// Local crate
match 2u32 as Bone {
    Bone::Tibia => println!("shin-specific logic"),
    Bone::Fibula => println!("fibia"),
    Bone::Spooky => println!("value not listed in upstream enum definition"),
    future_bone => println!("variant in future upstream enum definition"),
}
```

Initially, this will travel the `Spooky` path. However, when `Bone` updates and
adds a new `Femur = 2` variant, the `match` will go down the path `future_bone`.
The variant addition causing this behavior change is not considered a breaking
change; it is unique effect of what an open variant means.

- On an enum with 255 variants, this is allowed, but makes as much sense as putting `#[repr(transparent)]` and `#[non_exhaustive]` on an enum at the same time.

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
#[repr(u32)]
enum Shape {
    Square { width: f32 },
    Circle { radius: f32 },
    // error[E0740]: open enums cannot contain fields that may need dropping
    //  = note: this enum is open because it specifies an open variant
    //      Unknown = ..,
    //      ^^^^^^^^^^^^
    //  = note: a type is guaranteed not to need dropping when it implements
    // `Copy`, or when it is the special `ManuallyDrop<_>` type
    // help: when the type does not implement `Copy`, wrap it inside a
    // `ManuallyDrop<_>` and ensure it is manually dropped
    //      Path { points: std::mem::ManuallyDrop<Vec<(f32, f32)>> },
    Path { points: Vec<(f32, f32)> },
    Unknown = ..,
}
```

Open enums with fields are most useful in scenarios where the validity of the
value is not as important as its value remaining untouched, as well as binary
forwards compatibility with future versions of an enum.

```rust
#[repr(u32, transparent)]
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

Explain the proposal as if it was already included in the language and you were
teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should _think_ about the feature, and how it
  should impact the way they use Rust. It should explain the impact as
  concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or
  migration guidance.
- If applicable, describe the differences between teaching this to existing Rust
  programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section
should focus on how compiler contributors should think about the change, and
give examples of its concrete impact. For policy RFCs, this section should
provide an example-driven introduction to the policy, and explain its impact in
concrete terms.

# Reference-level explanation

TODO

It is marked as "universally non exhaustive". The open branch is not
necessary if the enum has a variant for every value of the underlying integer.

## Fielded enum layout

### Union-of-structs representation

An open variant on a `#[repr(Int)]` fielded `enum` now means: the enum must be
represented as a C-union of C-structs that each start with an integer newtype
that is `#[repr(transparent)]` over `Int`. This union also includes an
additional C-struct field used to represent all unrecognized variants.

The other fields of the structs are the payloads of the variants, used to store
the data. Here's an example - this definition:

```rust
#[repr(Int)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
    Unknown = ..,
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

#[repr(Int)]
enum MyEnumTag { A, B, C, D, Unknown = .. };

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

NOT FINISHED, TODO:

The more convenient but less efficient `(tag, union)` structure is also
available for stable specification as an open enum. An open variant in a
`#[repr(C, Int)]` fielded `enum` now means: the enum is a C-struct with two
fields, a `#[repr(Int)]` open enum containing each of the tag, and a C-union
with a field for every variant's data, including an extra empty field for
unrecognized variants.

So, this enum:

```rust
#[repr(C, Int, transparent)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
    Unknown = ..,
}
```

has equivalent layout to the following:

```rust
#[repr(C)]
struct MyEnumRepr {
    tag: MyEnumTag,
    payload: MyEnumPayload,
}

#[repr(Int)]
enum MyEnumTag { A, B, C, D, Unknown = .. };

#[repr(C)]
union MyEnumPayload {
   A: u32,
   B: MyEnumPayloadB,
   C: MyEnumPayloadC,
   D: (),
   Unknown: (),
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
- Fielded open enums cannot derive `Eq`, `Hash`, or `Ord`, since there is no
  reasonable default to interpret unknown variants.
- The [`mem::Discriminant`] of an open enum is equivalent to its `Int`
  representation.

[`mem::Discriminant`]: https://doc.rust-lang.org/core/mem/struct.Discriminant.html

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

## Integer-exhaustive enums

The open branch requirement specified throughout this RFC only applies to open enums that can represent unlisted variants. An enum that has a variant for every value of its underlying integer does not require an open branch, as it is exhaustive. In this case, specifying the `transparent` makes no difference in representation or behavior.

# Drawbacks

New:

- Matching against just an open variant is more expensive compared to other enum
  matches, since it has to check whether the integer is an unlisted value
  instead of being a specific value. This is less of an issue when it is used as
  the wildcard branch of an enum, since at that point it's unnecessary to check
  against the listed variants.

Old (previous draft, ignore):

- Open enums lose niche optimization - `Option<OpenEnum>` is always larger than
  `OpenEnum`, since the `Option` can't reuse unused discriminant values.
- Open enums require handling the "unknown" case in every `match`, though this
  is more of an intended feature.
- Making an enum open acts a lot like `#[non_exhaustive]` for users,
  except it also applies within the crate. This could be confusing for users
  who want to know the difference between this and `#[non_exhaustive]`.
- The existing macro ecosystem largely assumes enums are exhaustive, meaning
  they might need to specially consider the listed representation of the type
  when determining whether to emit an open branch or not for a `match`.
- `#[repr(transparent)]` does not _always_ imply that the value has the same
  niches as its inner non-zero-sized value. In nightly, `NonZeroU32` is
  `#[repr(transparent)]` over `u32` with additional nightly
  `rustc_layout_scalar_valid_range_start(1)` and
  `rustc_nonnull_optimization_guaranteed` attributes.
- Adding `#[repr(transparent)]` to an enum with `#[repr(u32)]` would be a
  breaking change, which may be surprising and is inconsistent with other uses of `#[repr(transparent)]`.
- `#[repr(transparent)]` now has two different meanings on an `enum`. The
  current (rare) usage is used for one variant enums, and requires that the
  variant's data follows the same rules as a transparent `struct`.
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

Discuss prior art, both the good and the bad, in relation to this proposal. A
few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature
  exist in other programming languages and what experience have their community
  had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us
whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not
on its own motivate an RFC. Please also take into consideration that rust
sometimes intentionally diverges from common language features.

# Unresolved questions

- Does the open variant have to be last?
- Should exhaustiveness checks in clippy require checking open variants
- Should it be illegal to name the open variant in an expression context,
  in order to avoid the `value != Enum::Unknown` footgun?

## Is this the right feature name/syntax?

This feature was previously called "wildcard variants" due to the overlap in
concept with `match` wildcard branches, but this was changed due to the overlap
with `_` being called the "[wildcard pattern][wildcard-pattern]". We could use
this name and have `_` be the discriminant, but this seemed much less clear in
intention than `..` is. The author also considered "rest variant" to mirror `..`
in a pattern, which is called the "[rest pattern][rest-pattern]". However, `..`
is also called the "open range" when used in an expression context, which enum
discriminants are.

[rest-pattern]: https://doc.rust-lang.org/reference/patterns.html?highlight=rest#rest-patterns
[wildcard-pattern]: https://doc.rust-lang.org/reference/patterns.html?highlight=rest#wildcard-pattern

It could also be a modification of `#[]`.

### What about `#[non_exhaustive]`?

We could require that any open enum also specify `#[non_exhaustive]`, to act as
a large well-known signal to readers that the enum has non-exhaustive semantics
and could have variants added in the future (in this case, with no ABI change).
This RFC goes "one step" farther than `#[non_exhaustive]`: it's
non-exhaustive at a binary level, and so also requires same-crate users to treat
it as non-exhaustive. So, if we required this annotation, should it be a
variation on `#[non_exhaustive]` to make the crate-local effect clear?

## `derive(Clone, Copy)` on fielded open enums

TODO

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

# Future possibilities

## Custom discriminant values for fielded enums

Enums that carry data do not currently have any way to specify what their
discriminant values are - they always go up from 0. So, Rust could provide a
syntax to specify a specific discriminant value for enum variants with fields,
like:

```rust
#[repr(C, u32, transparent)]  // `transparent` optional
enum PacketData {
    // The discriminant for the `Tcp` variant is the value of `Protocol::Tcp`
    Tcp(TcpData) = Protocol::Tcp,

    // The discriminant for the `Udp` variant is the value of `Protocol::Udp`
    Udp(UdpData) = Protocol::Udp,
}
```

This would be valuable for users who want fine-tuned control over the
representation of their enums, especially for cases where the tag is also a
well-known constant like a protocol ID.

## Extracting the integer value of the discriminant for fielded enums

A fielded enum with `#[repr(Int)]` and/or `#[repr(C)]` is guaranteed to have its
discriminant values starting from 0. However, for any given value of that enum,
there's no built-in way to extract what the integer value of the discriminant is
safely. The unsafe mechanism is `(&thenum as *const _ as *const Int).read()`.
For open fielded enums, this would be even more valuable, since the discriminant
could be entirely unknown and the programmer may want to know its value. Perhaps
an extension to [`mem::Discriminant`]?

Think about what the natural extension and evolution of your proposal would be
and how it would affect the language and project as a whole in a holistic way.
Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal. Also consider how
this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC
you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply
state that you cannot think of anything.

Note that having something written down in the future-possibilities section is
not a reason to accept the current or a future RFC; such notes should be in the
section on motivation or rationale in this or subsequent RFCs. The section
merely provides additional information.
