- Feature Name: (`enum-underscore-variants`)
- Start Date:
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

Enable ranges of enum discriminants to be reserved ahead of time, requiring
all users of that enum to consider those values as valid. This includes within
the declaring crate.

The `_ = RANGE` syntax in an enum definition declares that enum values with
discriminants in `RANGE` represent _reserved_ variants of that enum. It is sound
to construct these reserved variants with `unsafe`, and to handle them over FFI.
If there is no invalid discriminant for an enum, it becomes an _open enum_. If
it is [unit-only], it can then be converted from its explicit underlying
integer.

[unit-only]: https://doc.rust-lang.org/reference/items/enumerations.html#r-items.enum.unit-only

# Motivation

Enums in Rust have a _closed_ representation, meaning the only valid
representations of that type are the variants listed, with any violation of this
being [Undefined Behavior][ub]. This is the right default for Rust, since it
enables niche optimization and ensures values have a known state, limiting
unnecessary or dangerous code paths.

[ub]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

However, a closed enum is not always the best choice for systems programming.
The issue lies with compatibility between existing binaries.
There are many cases in which code is expected to handle non-yet-known
enum values as a non-error.

Consider a complex system that initially uses this `TaskState` enum to communicate:

```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[non_exhaustive]
/// `TaskState` v1
pub enum TaskState {
    Stopped = 0,
    Running = 1,
}
```

`non_exhaustive` is specified for forwards compatibility, since it should be a
non-breaking change for variants to be added to `TaskState`. This works by
requiring foreign crates to include a wildcard branch when `match`ing.
Once a new `Paused` variant is added to `TaskState`, any code that previously compiled
when using the `TaskState` will continue to do so. However, if any part of the
system is _not_ recompiled, that old code will see the `Paused` variant as invalid.

```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[non_exhaustive]
/// `TaskState` v2
pub enum TaskState {
    Stopped = 0,
    Running = 1,
    // A new valid discriminant for `TaskState` has been introduced!
    Paused = 2,
}
```

What if it isn't feasible to recompile **every** part of the system that uses
the enum in order to avoid the breaking change?

```rust
/// `TaskState` v1 using reserved enum discriminants instead of `non_exhaustive`.
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum TaskState {
    Stopped = 0,
    Running = 1,
    // There are reserved variants for the rest of the discriminants:
    // The `_` resembles a wildcard seen when `match`ing.
    _ = ..,
}
```

If every binary is using this definition, it is not an breaking change for
existing binaries using this definition to add `Paused = 2`.
The `_ = ..` has required _every_ exhaustive `match` of `TaskState`,
including in the defining crate, to handle the case where it's not one of the
currently-named variants.

## Protobuf

Protocol Buffers (Protobuf), a language-neutral serialization mechanism, is
designed to be forwards and backwards compatible when extending a schema.
Initially, it defined all of its enums as closed. However, this caused confusing
and often incorrect behavior with `repeated` enums, and so the `proto3` syntax
[switched to open enums][protobuf-history]. Handling unknown values
transparently comes up often in microservices where incremental rollouts cause
schema version skew.

[protobuf-history]: https://protobuf.dev/programming-guides/enum/#history

Protobuf generates code for target languages from a schema. On C++, it can
directly generate an `enum` - C++ enums are open since it's valid to
`static_cast` an `enum` from its backing integer. However, on Rust, the current
implementation simulates an open enum by using an integer newtype with
associated constants for each variant.

While this allows Protobuf enums in Rust to be used _mostly_ like enums, this is
a suboptimal experience:

- It is arduous to read the generated definition - the variants are inside
  of an `impl` instead of next to the name. It hides the type's nature as an enum.
- It's invalid to `use` the pseudo-variants like with `use EnumName::*`.
- Rust is a systems language that can move data around efficiently, and so
  first-class support for named integers is valuable for embedded programmers.
- Code analysis and lints specific to enums are unavailable.
  - No "fill match arms" in rust-analyzer.
  - The [`non-exhaustive patterns` error][E0004] lists only integer values,
    and cannot suggest the named variants of the enum.
  - The unstable `non_exhaustive_omitted_patterns` lint has no way to
    work with this enum-alike, even though treating it like a `non_exhaustive` enum
    would be more helpful.

[E0004]: https://doc.rust-lang.org/stable/error-index.html#E0004

If Protobuf instead declared generated Rust enums with a `_ = ..` variant,
users could have a first-class enum experience with compatible open semantics.

## C interop

A closed `#[repr(C)]` field-less `enum`s is [hazardous][repr-c-field-less]
to use when interoperating with C, mostly because it is so easy to
trigger Undefined Behavior when unknown values appear. In C, it is idiomatic
to do an unchecked cast from integer to enum. So, even if one ensures
that the C and Rust libraries are compiled at the same time, they must
also audit the C source to ensure that unknown values cannot be exposed
to Rust.

[repr-c-field-less]: https://doc.rust-lang.org/reference/type-layout.html#reprc-field-less-enums

With underscore variants, the current guidance surrounding sharing enums
with C can thus be simplified greatly:
add a `_ = ..` variant and UB from invalid values aren't a concern.

Bindgen has [multiple ways][bindgen-enum-variation] to generate Rust
that correspond to a C enum, the default being to define a series of
`const` items. A future version of bindgen could use this feature to
add a `_ = ..` variant to a Rust `enum` instead of a less-effective
`non_exhaustive` attribute.

[bindgen-enum-variation]: https://docs.rs/bindgen/0.72.1/bindgen/enum.EnumVariation.html

## Flag enums

> Q: Should this be included? It's more of a nice side-effect than a motivation IMO

Bitflag-style enums must allow more discriminants than are named in order to
represent combinations of known values. The `bindgen` and `bitflags` libraries
use integer newtypes, but they could instead use a first-class enum.

> Q:
> Perhaps a lint should trigger when `|` is used in a pattern with a
> non-integer type that defines `BitOr`?
> It's weird that `!matches!(Foo::A | Foo::B, Foo::A | Foo::B)`.
> Making it easier to define first-class bitflag enums seems like it'd make
> it easier to fall into this trap. This confusion still occurs even with a
> regular enum or `derive(PartialEq, Eq)` integer newtype with associated
> constants.

## Dynamic Linking

Dynamically linked libraries, Rust or otherwise, are prone to ABI compatibility breakage.

Ensuring ABI compatibility when extending a library requires extra care. While
`non_exhaustive` grants API compatibility as variantqs are added, it
[does _not_ provide ABI compatibility][non-exhaustive-ub].
By reserving discriminants for future extensions to an enum, libraries can choose
to remain ABI forwards-compatible as new variants are added.

Projects like Redox and relibc would use this feature for this reason among
others.

[non-exhaustive-ub]: https://github.com/rust-lang/rust-bindgen/issues/1763

## Embedded syscalls

TockOS is an embedded OS with a separate user space and kernel space. Its
syscall ABI defines that kernel error codes are between 1 and 1024. It's highly
desirable to keep the `0` niche available for `Result<(), ErrorCode>`, so the
user space library defines an [`ErrorCode` enum][libtock-errorcode] with 14
normal variants and 1010 "reserved" variants that will eventually be renamed.
This has drawbacks:

- It clutters the enum definition.
- rust-analyzer's "Fill match arms" inserts a new match arm for each of the
  reserved names, even though a single wildcard branch would be more
  appropriate.
- Since the reserved discriminants have named variants, there's nothing
  preventing users from using the reserved name. There is no perfect way to
  claim a reserved discriminant without breaking the API.
  - Declaring an associated `const` is the way to prevent an API breakage.
  - Moving a reserved variant name like `N00014` to a `deprecated` associated
    `const` is better for readability, but breaks any user that wrote
    `use ErrorCode::N00014`.
  - Declaring the new variant name as an associated `const` is harder to read,
    doesn't interact with code analyzers, and doesn't let users write
    `use ErrorCode::NewVariant`.

[libtock-errorcode]: https://github.com/tock/libtock-rs/blob/master/platform/src/error_code.rs#L30-L33

## zero-copy deserialization

A common pattern on embedded systems is to read data structures directly from
a `[u8]`, facilitated by libraries like
[`zerocopy`][zerocopy-frombytes-derive] or
[`bytemuck`][bytemuck-checkedbitpattern]. In order to do this, the bytes in
flash must always be validated to be one of the known discriminants.

This scales poorly for performance and code bloat as more enums are added to be
deserialized in a message. It is more flexible to defer wildcard branches for
unknown discriminants to the point when the enum is `match`ed on, rather than
up-front during deserialization. When these checks are undesirable, ergonomics
must be sacrificed for compatibility and performance by using an integer
newtype.

[bytemuck-checkedbitpattern]: https://docs.rs/bytemuck/latest/bytemuck/checked/trait.CheckedBitPattern.html
[zerocopy-frombytes-derive]: https://docs.rs/zerocopy/0.6.1/zerocopy/derive.FromBytes.html

## Restricted range integers

> Q: Can I show clear motivation for the value of restricted range integers in
> the first place?

Underscore variants can be used to define integers that are statically
restricted to a particular range, including with niches.

```rust
macro_rules! make_ranged_int {
    ($name:ident : $repr:ty; $($range:tt)*) => {
        #[repr($repr)]
        enum $name {
            _ = $($range)*,
        }
        impl TryFrom<$repr> for $name {
            type Error = ();
            fn try_from(val: $repr) -> Result<$name, ()> {
                match val {
                    // SAFETY: `val` is a valid discriminant for `$name`
                    $($range)* => Ok(unsafe { mem::transmute(val) }),
                    _ => Err(()),
                }
            }
        }
        impl From<$name> for $repr {
            fn from(val: $name) -> $repr {
                val as $repr
            }
        }
    };
}
make_ranged_int!(FuelLevel: u32; 0..=100);

assert!(size_of::<FuelLevel>() == size_of::<Option<FuelLevel>>());
assert_eq!(FuelLevel::try_from(10).unwrap() as u32, 10);
assert!(FuelLevel::try_from(21).is_err());
```

With other extensions, this could even be generic:

```rust
trait EnumDiscriminant {
    type Ranged<const RANGE: Range<Self>>;
}

impl EnumDiscriminant for u32 {
    type Ranged<const RANGE: Range<Self>> = RangedU32<RANGE>;
}

#[repr(u32)]
enum RangedU32<const RANGE: Range<u32>> {
    _ = RANGE,
}

type Ranged<T, const RANGE: Range<T>> = <T as EnumDiscriminant>::Ranged<RANGE>;

type FuelLevel = Ranged<u32, 0..=100>;
```

# Guide-level explanation

Enums have a _closed_ representation by default, meaning that any enum value
must be represented by one of the listed variants. Constructing any enum value
with an unassigned discriminant is immediate [Undefined Behavior][ub]:

```rust
#[repr(u32)]     // Fruit is represented with specific discriminants of `u32`.
enum Fruit {
    Apple,       // Apple is represented with 0u32.
    Orange,      // Orange is represented with 1u32.
    Banana = 4,  // Banana is represented with 4u32.
}
// Undefined Behavior: 5 is not a valid discriminant for `Fruit`!
let fruit: Fruit = unsafe { core::mem::transmute(5u32) };

// Rust utilizes these invalid discriminants for compiler-dependent
// optimization:
assert_eq!(mem::transmute(Option::<Fruit>::None, 2u32));
```

However, by declaring an **underscore variant**, the discriminant `5` is
_reserved_ and becomes sound to transmute from.

```rust
#[repr(u32)]     // An explicit repr is required to declare an underscore variant.
enum Fruit {
    Apple,       // Apple is represented with 0u32.
    Orange,      // Orange is represented with 1u32.
    Banana = 4,  // Banana is represented with 4u32.
    _ = 5,       // Some future variant will be represented with 5u32.
}
// SAFETY: 5 is a reserved discriminant for `Fruit`.
let fruit: Fruit = unsafe { core::mem::transmute(5u32) };

// `fruit` is not any of the named variants.
assert!(!matches!(fruit, Fruit::Apple | Fruit::Orange | Fruit::Banana));

// These are both rejected: an underscore variant can't construct reserved
// discriminants or patttern match on them.
// assert!(!matches!(fruit, Fruit::_));
// let fruit = Fruit::_;
```

By introducing this special variant, all users of `Fruit` must include a
wildcard branch when `match`ing, including within the declaring crate. Think of
the `_ = 5` as declaring that "discriminant `5` goes in the `_` branch" when
`match`ing. There's no safe way to construct a `Fruit` from a `5`, but it can be
`transmute`d or received over FFI.

```rust
match fruit {
    Fruit::Apple | Fruit::Orange | Fruit::Banana => println!("Known fruit"),
    // Must be included, even in the crate that defines `Fruit`.
    x => println!("Unknown fruit: {}", x as u32),
}
```

An underscore variant accepts a range as its discriminant expression, which
reserves each discriminant in the range.

```rust
#[repr(u32)]     // Fruit is represented with specific discriminants of `u32`.
enum Fruit {
    Apple,       // Apple is represented with 0u32.
    Orange,      // Orange is represented with 1u32.
    Banana = 4,  // Banana is represented with 4u32.
    _ = 5..=10,  // 5 through 10 inclusive are reserved discriminants for `Fruit`.
}
// SAFETY: 7 is a reserved discriminant for `Fruit`
let fruit: Fruit = unsafe { core::mem::transmute(7u32) };
```

By using `..` as an underscore variant range, all bit patterns for the enum
become valid. It is now an _open enum_ and can be constructed from its
underlying representation via `as` cast:

```rust
#[derive(PartialEq, PartialOrd)]
#[repr(u32)]     // Fruit is represented by any `u32` - it is an *open enum*.
enum Fruit {
    Apple,       // Apple is represented with 0u32.
    Orange,      // Orange is represented with 1u32.
    Banana = 4,  // Banana is represented with 4u32.
    _ = ..,      // The rest of the discriminants in `u32` are reserved.
}
// Using an `as` cast from `u32`.
let fruit = 3 as Fruit;

// Does not match any of the known variants.
assert!(!matches!(fruit, Fruit::Apple | Fruit::Orange | Fruit::Banana));

// `fruit` preserves its value casting back to `u32`.
assert_eq!(fruit as u32, 3);

// derive(PartialOrd, PartialEq) works by discriminant as usual:
assert!(5 as Fruit > fruit);
assert!(3 as Fruit == fruit);
assert!(1 as Fruit == Fruit::Orange);

// error: incompatible cast: `Fruit` must be cast from a `u32`
// help: to convert from `isize`, perform a conversion to `u32` first:
//         let fruit2 = u32::try_from(5isize).unwrap() as Fruit;
let fruit2 = 5isize as Fruit;
```

This open enum is much like a `struct Fruit(u32)`, except it is treated as
an enum by IDEs and developers.

## Interaction with `#[non_exhaustive]`

An enum declared both `non_exhaustive` and with an underscore variant is
rejected. On a field-less enum, it is not a breaking change to replace a
`#[non_exhaustive]` declared on the enum with a contained underscore variant.
Underscore variants and `#[non_exhaustive]` both declare that future variants of
an enum may be added as the type evolves.

`non_exhaustive` affects API compatibility:

- It is flexible in how new variants are represented.
- It does _not_ affect what discriminants are currently valid to represent.
- Crates must be recompiled to use new enum variants.
- It affects _only_ foreign crates.

By contrast, an underscore variant affects API _and_ ABI compatibility:

- It reserves specific ranges of discriminants.
- These reserved discriminants are valid to represent without naming the future
  variants that use them.
- Crates can manipulate these reserved enum variants without recompilation.
- It affects all crates, including the declaring one.

For enums that have relevant discriminant values, an underscore variant is thus
likely the better choice. This is often the case for enums declaring an explicit
`repr`.

# Reference-level explanation

## Underscore variants

An **underscore variant** is an enum variant with `_` declared as its name. An
underscore variant reserves one or many discriminants as valid for an enum,
representing not-yet-named variants. It does not declare an identifier scoped
under the enum name, unlike named variants. `EnumName::_` remains an invalid
expression and pattern.

It is valid to transmute to an enum type from a reserved discriminant integer.

An underscore variant may be specified more than once on the same enum.
It is valid to reserve multiple ranges of discriminants.
Those ranges may be discontiguous.

An explicit `repr(Int)` is required on an enum to declare an underscore variant.
`Int` is one of the primitive integers or `C`. If it is `C`, then `Int` below
is `isize`. An underscore variant must specify a discriminant expression with
one of these types:

- `Int`
  - Reserves a particular discriminant value.
  - Cannot duplicate any existing discriminants.
- `start..end` (`core::ops::Range<Int>`) or\
  `start..=end` (`core::ops::RangeInclusive<Int>`)
  - Reserves every discriminant value in the range.
  - The range must be non-empty.
  - Must not overlap with any other existing discriminants for the enum.

    ```rust
    #[repr(u8)]
    // error: discriminant value `10` assigned more than once
    enum Foo {
        X = 0,
        _ = 1..=10,
        Y = 10,
    }

    // error: discriminant values `10..=14` assigned more than once
    enum Bar {
        X = 0,
        _ = 1..20,
        Y = 20,
        _ = 10..15,
    }
    ```

- `..` (`core::ops::RangeFull`)
  - Reserves the rest of the discriminants for `Int`.
    These are the discriminants that don't have a named variant.
  - When used, this is the only underscore variant allowed on the enum.

    ```rust
    // error: discriminant value `1` assigned more than once
    // help: an `_` variant assigned to `..` forbids other `_` variants
    #[repr(u8)]
    enum Foo {
        X = 0,
        _ = 1,
        Y = 2,
        _ = ..,
    }
    ```

  - This design allows `_ = ..` to simply make an enum open without
    consideration for named variants' discriminants.
  - There must be at least one discriminant available to reserve.
    An underscore variant cannot be specified on an enum that is already open:

    ```rust
    #[repr(u8)]  // #[non_exhaustive] is valid here though
    enum NamedU8 {
        James = 0,
        Fernando = 1,
        Sally = 2,
        /* every other u8 */
        Joe = 255,

        // error: there are no discriminants to reserve in `NamedU8`
        _ = ..,
    }
    ```

### `repr(C)` behavior

`repr(C)` enums have special semantics in Rust because the discriminant
expression type, `isize`, is not the same as the actual backing integer.
These enums are ordinarily backed by a `ffi::c_int`, but if any of the
assigned discriminants cannot fit, a larger backing integer is chosen that
can represent all of them.

The same rules apply for discriminants assigned to underscore variants:

```rust
#[repr(C)]
enum Small {
    X = 1,
    _ = 2..10,
}

enum Big1 {
    X = 1,
    _ = isize::MAX,
}

enum Big2 {
    X = 1,
    _ = 2,
    Y = isize::MAX,
}

// On x86_64-pc-windows-msvc
const _: () = assert!(
    size_of::<Small>() == 4 &&
    size_of::<Big1>() == 8 &&
    size_of::<Big2>() == 8
);
```

When `..` is used as a discriminant in a `repr(C)` enum, the set of
discriminants that is reserved is dependent on the other named
variants.

```rust
#[repr(C)]
enum SmallOpen {
    X = 0,
    // Reserves `c_int::MIN..0` and `1..=c_int::MAX`
    _ = ..,
}

#[repr(C)]
enum BigOpen {
    X = isize::MAX,
    // Reserves `isize::MIN..isize::MAX`
    _ = ..,
}

// On x86_64-pc-windows-msvc
const _: () = assert!(
    size_of::<SmallOpen>() == 4 &&
    size_of::<BigOpen>() == 8
);
```

This behavior means that it is sound to expose a C enum defined like this:

```c
enum Foo {
    Name1 = Value1,
    Name2 = Value2,
    // etc.
};
```

as this Rust enum, regardless of the discriminant values assigned:

```rust
#[repr(C)]
enum Foo {
    Name1 = Value1,
    Name2 = Value2,
    // etc.

    // The rest of the discriminants for an enum with the named variants
    // are reserved and valid. Unchecked casts can't invoke UB.
    _ = ..,
}
```

On platforms where

### Grammar changes

[EnumVariant] is extended to allow an underscore instead of a variant's name:

```text
EnumVariant ->
  OuterAttribute* Visibility?
  (IDENTIFIER | `_`) ( EnumVariantTuple | EnumVariantStruct )?
  EnumVariantDiscriminant?
```

[EnumVariant]: https://doc.rust-lang.org/reference/items/enumerations.html#grammar-EnumVariant

### No field data

This RFC only defines adding underscore variants to field-less enums, leaving
this as future work.

### Compatibility

Given enum versions A and B with some change between them:

- A change is forwards-compatible if a library designed for enum version A can
  use A or B.
- A change is backwards-compatible if a library designed for enum version B can
  use A or B.
- A change is fully-compatible if it is both forwards and backwards compatible.
- A change is API compatible if the change does not affect static compilation
  using a single enum source, either A or B.
- A change is ABI compatible if the change does not affect dynamically linked
  libraries compiled using enum versions A and B.

It is a fully-compatibile change to:

- Add a variant to an enum using a discriminant that was previously reserved.
- When doing the above, removing the last underscore variant may cause warnings
  for unused code in client libraries. This can be avoided by adding
  `#[non_exhaustive]` to the enum.

It is an API fully-compatible and ABI backwards-compatible change to:

- Replace `#[non_exhaustive]` on an enum with an underscore variant.
  - This may require changes to the defining crate to add wildcard branches.
- Add another reserved discriminant, if an underscore variant already exists on
  the enum.

It is an API and ABI backwards-compatible change to:

- Add an underscore variant to an enum without `#[non_exhaustive]` or another
  underscore variant. The same caveat regarding unused wildcard branches
  applies.

### `non_exhaustive`

The `non_exhaustive` attribute on enums and underscore variants are mutually exclusive:

```rust
#[non_exhaustive]
#[repr(u8)]
enum Color {
    Red = 0,
    Green = 1,
    /// error: An `_` variant cannot be specified on a `non_exhaustive` enum.
    // help: remove the `#[non_exhaustive]`
    _ = 2,
}
```

An underscore variant is more impactful than `non_exhaustive`, since it affects
the declaring crate - the enum is "universally non-exhaustive".

## Open enum conversion

An _open enum_ is defined as an enum for which every initialized bit pattern
is valid.

- An open enum always has an explicit `repr` backing integer.
- An enum is open if every discriminant value for that integer is associated
  with a named variant or is reserved with an underscore variant.
- Open enums may be `as` cast from their backing integer: `2u8 as Color`.
- Casting from other integer types is rejected.
- If an expression with the `{integer}` inference variable type is used as the
  source for an `as` cast to an open enum, it is uniquely constrained to the
  backing integer type.

    ```rust
    #[repr(u8)]
    enum Foo { _ = .. }
    let x = 10;

    // `x` must be a `u8` to be cast to `Foo`
    let _ = x as Foo;
    
    // error: mismatched types, expected `u32`, found `u8`
    // let _: u32 = x;
    ```

## Interaction with the standard library

- `derive(Debug)` formats as `EnumName(X)` when `X` is a reserved discriminant.
  A `Debug` format changing is not considering an API-breaking change.
- The derives `Clone`, `Copy`, `Default` `Eq`, `Hash`, `Ord`, `PartialEq`,
  and `PartialOrd` are unaffected by underscore variants on a field-less enum.
  They all operate on discriminants, and this includes reserved discriminants.

# Drawbacks

The mutual-exclusion with `non_exhaustive` despite having similar motivations
could be confusing to explain to new users. Every new feauture in Rust is
another thing to maintain and for users to learn. Rust has not put significant
efforts towards ABI compatibility in language constructs in the past.

Bit flag enums can now be first-class `enums` with `_ = ..`. Whether this is a
boon or a drawback is TBD.

> Q: More drawback ideas are welcome. Are bit flag enums here good?

# Rationale and alternatives

Underscore variants enable a large range of discriminants to be reserved for an
enum, whether it's all or some of them. `rustc_layout_scalar_valid_range_*`,
`NonZero`, and spelling out each discriminant are the only other ways to achieve
this in rustc today.

The open enum conversion from backing integer is an ergonomic benefit
that is made possible by underscore variants.

- The function-style constructor allows migrating from an integer newtype to a
  first-class enum with underscore variants as a non-breaking change.
- The `as` cast from discriminant integer mirrors the cast to it.

## Why a language extension? Why not just use an integer newtype or macro?

The best way to write a field-less open enum in Rust today is the "newtype enum"
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
except there must _always_ be a wildcard branch for handling unknown values.
This syntax also provides grouping of related values and associated methods, an
advantage over module-level `const` items.

However, this pattern has some distinct disadvantages, as described in the
Motivation section.

## As an `enum` attribute

An enum could be made open by specifying it as part of its `repr`:

```rust
#[repr(open, u8)]  // requires an explicit `repr(Int)`
enum Color {
    Red,
    Blue,
    Black
}
use Color::*;
// or an unsafe `transmute`
assert!(!matches!(3u8 as Color, Red | Blue | Black));
```

This has the same interaction with `#[non_exhaustive]`. The drawbacks:

- It is not clear why a `repr` would affect `match`/`as` behavior, even though
  this does affect how it is valid to represent the type.
  - An alternative syntax is perhaps `#[non_exhaustive(repr)]` or `[open]`
    / `#[open(Range)]`. Both require a `repr(Int)` be specified.
- Allowing for a reservation of particular ranges instead of a full opening
  could be done with extra detail:

  ```rust
  #[repr(open(3..=10), u8)]  // reserve discriminants 3..=10 only
  enum Color {
      Red,
      Blue,
      Black
  }
  ```

- It's not as clear what the attribute does, in contrast to the `_ = ..` syntax
  mirroring known concepts: `_` means "unnamed/wildcard", and `..` means "the
  rest" as the discriminants.

## Discriminant ranges for named variants instead of underscore variants

What if instead this were valid?

```rust
enum IpProto {
    Tcp = 6,
    Udp = 17,
    Other = ..,
}
```

This is not mutually exclusive with underscore variants, but this RFC chooses to
leave reserved ranges of discriminants as anonymous to keep the feature simple.

- It is ambiguous what value should be chosen when `IpProto::Other` is used in
  an expression.
- Even with an arbitrary rule to choose a discriminant, a consistently
  performant `derive(PartialEq)` that compares discriminants instead of ranges
  of discriminants will result in
  `matches!(o, IpProto::Other) && o != IpProto::Other`.
  - A reasonable but less useful alternative is to reject expression usage
    as well as `derive(PartialEq)`.
- If discontiguous ranges are allowed as above, the performance of
  `matches!(o, EnumName::Variant)` gets worse as the number of variants grows.
- Adding an `Icmp = 1` variant affects `matches!(1 as IpProto, IpProto::Other)`.
- A `derive` can be used to determine whether an enum's discriminant is assigned
  to a named variant.
- Anonymous discriminant values are useful on their own for enum evolution.

This can be left as future work for the language.

## `..` at the end

```rust
#[repr(u8)]
enum IpProto {
    Tcp = 6,
    Udp = 17,

    // "the rest of the variants exist"
    ..
}
```

- This is less flexible than `_ = ..`, and is awkward to restrict to smaller
  or discontiguous ranges.
- This resembles the [rest pattern] more than the [full range expression] that
  discriminants are assigned to and the [wildcard pattern] that it requires.

[full range expression]: https://doc.rust-lang.org/reference/expressions/range-expr.html#grammar-RangeFullExpr
[rest pattern]: https://doc.rust-lang.org/reference/patterns.html?#rest-patterns
[wildcard pattern]: https://doc.rust-lang.org/reference/patterns.html?#wildcard-pattern

## An "other" variant carries unknown discriminants like a tuple variant

An alternative way to specify a field-less open enum could be to write this:

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

However, this has some problems. For one, it's peculiar for a tuple variant
syntax to not carry a payload, but a discriminant. Also, this could lead to surprising
behavior with pattern matching, unless the match semantics are changed greatly:

```rust
if let IpProto::Other(x) = IpProto::Other(6) {
    // This branch is not taken, since it's actually an `IpProto::Tcp`!
}
```

This could do a check that the provided integer value isn't a known variant,
but that generates an implicit panic for what looks like a simple enum
construction. Instead, to get this behavior with this RFC's proposed syntax, the
author could use a third-party derive to check against the named variants:

```rust
#[repr(transparent, u32)]
#[derive(IsNamedVariant)]
enum IpProto {
    Tcp = 6,
    Udp = 17,
    _ = ..,
}

assert!(!(3u32 as IpProto).is_named_variant());
assert!((6u32 as IpProto).is_named_variant());
```

# Prior art

_Open_ and _closed_ enums are [pre-existing industry terms][acord-xml].

## Enum openness in other languages

- C++'s [scoped enumerations][cpp-scoped-enums] and C enums are both open
  enums.
- C# uses [open enums][cs-open-enums], with a [proposal][cs-closed-enums] to
  add closed enums for guaranteed exhaustiveness.
- Java uses closed enums.
- [Protobuf][protobuf-enum] uses closed enums with the `proto2` syntax, treating
  unlisted enum values as unknown fields, and changed the semantics to open
  enums with the `proto3` syntax. This was in part because of lessons learned
  from protocol evolution and service deployment as described above.

> Q: How much further should this section be extended?

[acord-xml]: https://docs.oracle.com/cd/B40099_02/books/ConnACORDFINS/ConnACORDFINSApp_DataType10.html
[cpp-scoped-enums]: https://en.cppreference.com/w/cpp/language/enum#Scoped_enumerations
[cs-open-enums]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/enum#conversions
[cs-closed-enums]: https://github.com/dotnet/csharplang/issues/3179
[protobuf-enum]: https://developers.google.com/protocol-buffers/docs/reference/cpp-generated#enum

## Other crates that use open enums

Users today are simulating open enums with other language constructs, but it's
a suboptimal experience:

- [open-enum], written by the author of this RFC. It's a procedural
  macro which converts any field-less `enum` definition to an equivalent newtype
  integer with associated constants.
- Bindgen is [aware of the problem][bindgen-ub] with FFI and closed enums, and
  avoids creating Rust enums from C/C++ enums because of this. It provides an
  option for newtype enums directly.
- ICU4X uses newtype enums for [certain properties][icu4x-props] which must be
  forwards compatible with future versions of the enum.
- OpenTitan's [`with_unknown!`] macro also uses this pattern to create
  "C-like enums".
- `winapi-rs` defines an [`ENUM`][winapi-enum] macro which generates plain
  integers for simple `enum` definitions.

The [`newtype-enum` crate][newtype-enum-crate] is an entirely different pattern
than what is described here.

[bindgen-ub]: https://github.com/rust-lang/rust/issues/36927
[icu4x-props]: https://github.com/unicode-org/icu4x/blob/ff1d4b370b834281e3524118fb41883341a7e2bd/components/properties/src/props.rs#L56-L106
[newtype-enum-crate]: https://crates.io/crates/newtype-enum
[open-enum]: https://crates.io/crates/open-enum
[`with_unknown!`]: https://github.com/lowRISC/opentitan/blob/06584dc620c633e88631f97f1fc1e22c1980c21c/sw/host/ot_hal/src/util/unknown.rs#L7-L48
[winapi-enum]: https://github.com/retep998/winapi-rs/blob/77426a9776f4328d2842175038327c586d5111fd/src/macros.rs#L358-L380

## `abi_stable`

[`abi_stable::NonExhaustive`] uses an associated type to hold a typed raw
discriminant for an enum. It is not ergonomic to `match` on discriminant values
directly, but another macro could improve this.

[`abi_stable::NonExhaustive`]: https://docs.rs/abi_stable/0.11.3/abi_stable/nonexhaustive_enum/struct.NonExhaustive.html

> Q: Should this mention [pattern types](https://github.com/rust-lang/rust/pull/107606)?

# Unresolved questions

- Should an underscore variant instead _require_ `#[non_exhaustive]`, instead
  of forbidding it? This would make lints that apply to `non_exhaustive` more
  obviously correct. It'd apply to `match`es in defining crate as well, which
  `non_exhaustive` is defined today not to do.
  - Consider this enum:

    ```rust
    #[repr(u8)]
    #[non_exhaustive]
    enum OpenEnum {
        X000 = 0,
        X001 = 1,
        // XNNN = N,
        X254 = 254,
        _ = 255,
    }
    ```

    When adding `X255`, the `non_exhaustive` should also be removed.
    As of today, an open enum gives no warning if it is `non_exhaustive`, even
    though it would necessarily be a breaking change to add a new variant by
    changing the `repr`.
- For enums with underscore variants, should the following lints apply?
  - [`non_contiguous_range_endpoints`] helps with off-by-one errors and
    `start..end` discriminant ranges.
  - [`non_exhaustive_omitted_patterns`] could apply to enums with underscore
    variants, but for defining-crate users as well. Should it have a different
    name?
- Should non-empty or `start > end` ranges be allowed for underscore variants?
  - Handling non-literal discriminant expressions in macros would benefit from
    this only being a warning instead of an error.
- Should other range types be supported? `start..` seems marginally
  useful, but also confusing especially with the more special behavior of `..`.
- Is "wildcard variant" a better name for `_ = <range-or-integer>` than
  "underscore variant"?

[`non_contiguous_range_endpoints`]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_lint/builtin/static.NON_CONTIGUOUS_RANGE_ENDPOINTS.html
[`non_exhaustive_omitted_patterns`]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_lint_defs/builtin/static.NON_EXHAUSTIVE_OMITTED_PATTERNS.html

# Future possibilities

> Q: Are pattern types appropriate to mention here?

## Discriminant ranges for named variants

A future extension could allow for named variants to specify ranges as
discriminants. This bikeshed syntax avoids many of the drawbacks in the
related Alternatives section above.

```rust
#[repr(u8)]
enum Color {
    Red = 0,
    Green = 1,
    // Must specify a non-overlapping range,
    // including if `..` is the discriminant.
    Unknown = 2..=u8::MAX,
}

// This is fine.
assert_eq!(Color::Red as u8, 0);

// error: ambiguous discriminant for `Color::Unknown`
// help: specify a discriminant with `2 as Color::Unknown`
// let c = Color::Unknown;

// Use an `as` cast to construct `Color::Unknown` safely.
let c = 3 as Color::Unknown;
assert_eq!(c as u8, 3);

// error: invalid discriminant for `Color::Unknown`
// help: `Color::Unknown` has the discriminant range `2..=u8::MAX`
// let c = 0 as Color::Unknown;

// let d = 10;
// error: non-constant expression used for enum ranged variant cast
// let c = d as Color::Unknown;

// This is fine.
let c = const { 1 + 1 } as Color::Unknown;
```

## Underscore variants on enums with field data

Underscore variants on enums with field data would allow library authors to plan
for future ABI compatibility by reserving discriminants and data space for an
enum. This requires significantly more documentation and care regarding ABI
stability before this can be stabilized.

For example:

```rust
#[repr(u32)]
pub enum Shape {
    Circle { radius: f32 } = 0,
    Rectangle { width: f32, height: f32 } = 1,
    _ = 2..=10,
}
```

- This reserves discriminants `2..=10` as valid for the `Shape` enum. It's not
  an ABI-breaking change to add new variants with data to `Shape` using these
  discriminants, so long as it doesn't affect the layout of the `Shape`.
- `Drop` glue is forbidden for field data (for a similar reason as `union`).
- The payload bytes of `Shape` are treated as opaque and never as padding.

By putting field data in an underscore variant, `Shape` can specifically
reserve the size and alignment needed to hold future variants' fields:

```rust
#[repr(u32)]
pub enum Shape {
    Circle { radius: f32 } = 0,
    Rectangle { width: f32, height: f32 } = 1,

    // This reserves discriminants `2..=10` and the layout to hold a
    // thin pointer without breaking ABI. It's as if there were a variant
    // for `&'static ()` in the enum's internal `union`.
    _(&'static ()) = 2..=10,

    // Because of the above, it's not an ABI-breaking change to add this
    // variant since the layout won't be affected:
    // FromInfo { name: &'static ShapeInfo } = 2,
}
```

## Tuple-like syntax for `repr` enums

A very useful thing this RFC enables is that replacing this:

```rust
// The "newtype integer enum" pattern.
#[derive(PartialEq, Eq)]
pub struct Color(u32);
impl Color {
    const Red: Color = Color(0);
    const Blue: Color = Color(1);
    const Green: Color = Color(2);
}
```

with this:

```rust
#[derive(PartialEq, Eq)]
#[repr(u32)]
pub enum Color {
    Red,
    Blue,
    Green,
    _ = ..,
}
```

is a non-breaking change for client crates.

However, if the library initially exposed the discriminant field as `pub`, as
`bindgen`, `icu4x`, and `open-enum` do, then the migration to an open `enum`
requires that `Color(discriminant)` and `color.0` also function as originally.

These each have their own independent utility:

### Tuple constructor

The enum name is a constructor `fn(Repr) -> Enum`:

```rust
assert_eq!(Color(1), Color::Blue);
assert!(
    [0, 3, 2].map(Color),
    matches!([Color::Red, _, Color::Green])
)
```

- This is valid for any open enum, the same as the `as` cast from integer.
- This mirrors the `derive(Debug)` format, is ergonomic, and is clear at
  callsite. Thus it may be worth adding to Rust even if `.0` isn't.
- When should one prefer the constructor over the `as` cast? Always?

### Discriminant field access

`.0` provides direct access to the discriminant value:

```rust
let mut c = Color::Blue;
assert_eq!(c.0, 1);
c.0 += 1;
assert!(matches!(c, Color::Green));
assert_eq!(c.0, 2);
```

This is subjectively ugly and undiscoverable syntax to access the discriminant
of an `enum`. One possibility: when introduced, treat as deprecated and throw a
warning to recommend a better syntax than `.0` but still allow the desired
non-breaking migration.

There are a few distinct advantages compared to `as` casting:

- It is possible to get a reference directly to the discriminant, which can
  be useful when performing lifetime-constrained zero-copy serialization.
- The type of `.0` is exactly the `repr`, and doesn't require the user
  specify a type to `as` cast to and possibly truncate. Currently, there's no
  language feature in Rust that does this - it requires a macro or codegen
  to guarantee. This can cause subtle bugs, especially for `repr(C)`:

  ```rust
  #[repr(C)]
  enum Oops {
      // On any platform where this is more than `c_int::MAX`.
      TooBig = 2_147_483_649,
  }
  assert_eq!(Oops::TooBig as core::ffi::c_int, -2_147_483_647);
  ```

  Instead, `.0` accesses the discriminant without fear of truncation:

  ```rust
  assert_eq!(Oops::TooBig.0, 2_147_483_649);
  // mismatched types, expected `i32`, got `i64`
  // let _: c_int = X::V.0;
  ```

This could be supported for _any_ enum with an explicit `repr(Int)` by having
closed enums be `unsafe` to mutate through `.0` - it's an [unsafe field].

```rust
#[repr(u32)]
enum X {
    A = 0,
    B = 1,
}
let mut x = X::A;
assert_eq!(x.0, 0);

// SAFETY: 1 is a valid discriminant for `X`
unsafe { x.0 += 1; }

assert!(matches!(x, X::B));
```

[unsafe field]: https://rust-lang.github.io/rust-project-goals/2025h2/unsafe-fields.html

## Extracting the integer value of the discriminant for fielded enums

A fielded enum with `#[repr(Int)]` and/or `#[repr(C)]` is guaranteed to have its
discriminant values starting from 0. However, for any given value of that enum,
there's no built-in way to extract what the integer value of the discriminant is
safely. The unsafe mechanism is `(&thenum as *const _ as *const Int).read()`.
For open fielded enums, this would be even more valuable, since the discriminant
could be entirely unknown and the programmer may want to know its value.

Perhaps this uses the same `.0` syntax as above, or an extension to
`mem::Discriminant`?
