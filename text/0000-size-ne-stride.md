- Feature Name: `prep_size_ne_stride`
- Start Date: 2020-05-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC is proposing the addition of intrinsics and `core` traits and functions to enable Rust programs to distinguish the size and stride of types in such a  manner that programs remain correct when faced with layout algorithms which themselves distinguish size and stride (see: [future-possibilities](#future-possibilities)).

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

The primary motivation for doing this is to support [Swift](https://swift.org) ABI interoperability.
Such interoperability was discussed and considered briefly in [this internals thread](https://internals.rust-lang.org/t/a-stable-modular-abi-for-rust/12347)

However, the Swift 5 layout algorithm is one which distinguishes size and stride of types.
In particular it lays out latter fields of structs in the trailing padding of the preceding field,
when such padding is sufficiently aligned.

Due to this it would presently be UB/unsound to import such structs because none of Rust, rustc, core, std, nor any user crates promise that such latter fields will not be clobbered by accesses to earlier fields, and in fact user crates cannot even express the requirements they would need to uphold, which is what this RFC intends to correct.

A possibly canonical example is this one ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=deeeba557daf53d574e6e9484d34de73)):
```Rust
use core::num::NonZeroU8;
use core::mem;

// #[repr(Swift_5)]
struct Inner(u16, u8);

// #[repr(Swift_5)]
struct Outer(Inner, NonZeroU8);


const REPR_SWIFT_5: bool = false;

fn foo(x: &mut Inner) {
    x.1 = 0;
}

fn main() {
    if REPR_SWIFT_5 {
        assert_eq!(mem::size_of::<Inner>(), 4);
        assert_eq!(mem::size_of::<Outer>(), 4);
    } else {
        assert_eq!(mem::size_of::<Inner>(), 4);
        assert_eq!(mem::size_of::<Outer>(), 6);
        
    }

    let nz = NonZeroU8::new(42).unwrap();
    let i = Inner(335, 65);
    let mut o = Outer(i, nz);
    let ro = &mut o;
    let ri = &mut ro.0;
    foo(ri);
    assert_eq!(ro.1, nz);
    
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

###

Most types in Rust, principally including both `#[repr(C)]` and `#[repr(Rust)]` types,
claim their trailing padding for themselves in their layouts:

```rust
struct A {
    x: u64,
    y: u8,
}

let size = std::mem::size_of::<A>();;
let align = std::mem::align_of::<A>();

assert_eq!(size % align, 0);

#[repr(C)]
struct B {
    x: u64,
    y: u8,
}

let size = std::mem::size_of::<B>();;
let align = std::mem::align_of::<B>();

assert_eq!(size % align, 0);
```

However, Rust reserves the right to add a `#[repr(...)]` for which the preceding code would fail because there exist some type layout algorithms, e.g. [Swift 5](https://swift.org)'s layout algorithm described in [Collapse trailing padding](https://github.com/rust-lang/rust/issues/17027) and [Separate size and stride for types](https://github.com/rust-lang/rfcs/issues/1397).

For this reason, Rust provides the `Strided` trait, and the `std::mem::store_size_of` and `std::mem::stride_of` functions.

The `Strided` trait is the trait which promises that the preceding code is both well formed and correct. Similar to `Sized`, `Strided` is an implicit `auto trait` which is implicitly assumed for all generic parameters and can be explicitly rejected using `T: ?Strided`.

The two functions take on the two options available for `std::mem::size_of` when the guarantees of Strided may fail to hold:
* `std::mem::store_size_of<T: ?Strided>` returns the number of bytes which may be stored when one has a `*mut T`.
* `std::mem::stride_of<T: ?Strided>` returns the minimal number of bytes between distinct well aligned `*const T`


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

###

## Interaction with `#[repr(transparent)]`:

It would be source breaking to have `#[repr(transparent)]` prevent the type from conforming to `Strided` on it's own.
However, `#[repr(transparent)]` types will not conform to `Strided` when the wrapped type does not conform to `Strided`.

## The following is added to `core::marker`:
```rust
/// Types which own their trailing padding.
///
/// This trait is implicitly assumed for all generic parameters,
/// but can be explicitly rejected using `T: ?Strided`.
#[cfg_attr(not(bootstrap), lang = "strided")]
#[unstable(feature = "prep_size_ne_stride", issue = "17027")]
pub auto trait Strided {
    // Empty.
}
```

## The following is added to `core::intrinsics`:
```rust
/// The store size of a type in bytes.
///
/// More specifically, this is the maximum number
/// of bytes which may be stored when one has a `*mut T`.
///
/// The unstabilized version of this intrinsic is
/// [`std::mem::store_size_of`](../../std/mem/fn.store_size_of.html).
#[cfg(not(bootstrap))]
pub fn store_size_of<T>() -> usize;

/// The store size of the referenced value in bytes.
///
/// More specifically, this is the maximum number
/// of bytes which may be stored when one has a `*mut T`.
///
/// see [`std::intrinsics::store_size_of`](fn.store_size_of.html)
#[cfg(not(bootstrap))]
pub fn store_size_of_val<T: ?Sized>(_: *const T) -> usize;
```

## The following is added to `core::mem`:
```rust
/// Returns the store size of a type in bytes.
///
/// More specifically, this is the maximum number
/// of bytes which may be stored when one has a `*mut T`.
/// When laid out in vectors and structures there may be
/// additional padding between elements.
#[unstable(feature = "prep_stride_ne_size", issue = "17027")]
#[rustc_const_stable(feature = "prep_stride_ne_size", since = "1.46.0")]
pub const fn store_size_of<T>() -> usize {
    #[cfg(bootstrap)]
    let ret = intrinsics::size_of::<T>();
    #[cfg(not(bootstrap))]
    let ret = intrinsics::store_size_of::<T>();
    ret
}

/// Returns the stride of a type in bytes.
///
/// More specifically, this is the minimum distance
/// in bytes between distinct well aligned `*const T`.
#[unstable(feature = "prep_stride_ne_size", issue = "17027")]
#[rustc_const_stable(feature = "prep_stride_ne_size", since = "1.46.0")]
pub const fn stride_of<T>() -> usize {
    #[cfg(bootstrap)]
    let ret = intrinsics::size_of::<T>();
    #[cfg(not(bootstrap))]
    let ret = {
        let size = intrinsics::size_of::<T>;
        let align = intrinsics::align_of::<T>;
        let remainder = size % align;
        if (size == 0) { return align };
        if (remainder == 0) { size } else { size + align - remainder }
    };
    ret
}

/// Returns the store size of a type in bytes.
///
/// More specifically, this is the maximum number
/// of bytes which may be stored when one has a `*mut T`.
/// When laid out in vectors and structures there may be
/// additional padding between elements.
///
/// This is typicality the same as `store_size_of::<T>()`,
/// but that requires `Sized`, and this does not.
#[unstable(feature = "stride_ne_size", issue = "17027")]
#[rustc_const_stable(feature = "prep_stride_ne_size", since = "1.46.0")]
pub fn store_size_of_val<T: ?Sized>(val: *const T) -> usize {
    #[cfg(bootstrap)]
    let ret = intrinsics::size_of_val(val);
    #[cfg(not(bootstrap))]
    let ret = intrinsics::store_size_of_val(val);
    ret
}
```

## The stride of ZSTs:

This RFC defines the stride in such a way that it is strictly positive.
This means that the stride of a ZST is equal to it's alignment, and not zero.

This is a debatable decision, which is discussed [below](#rationale-and-alternatives).

## The size of `char`:

This RFC enables types to have sizes which are not multiples of their alignments.
The only primitive which may benefit from this is `char`, which requires 21 bits, 
which may be stored in 3 bytes.

Note that while Swift does distinguish between `store_size` and `stride`, it's equivalent of `char`, `Unicode.Scalar` still has a `store_size` of 4.

Should Rust change `char` to be `!Strided` and have a `store_size_of` 3 bytes?
This is an [unresolved-question](#unresolved-questions).

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?
See [Huan Wilson](https://github.com/huonw)'s comment [here](https://github.com/rust-lang/rfcs/issues/1397#issuecomment-213311508).
> My feeling is that inhibiting writing a 3-byte struct like (i16, bool) as a single 4 byte write everywhere (i.e. whether or not it is actually nested like ((i16, bool), bool)) is a larger downside than compressing any structs that actually look like that.

This is added complexity to the standard library, which may not be considered warranted.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?

    This RFC enables Rust programs to handle types which have sizes which are
    distinct from their strides, without requiring rustc to handle such types.
    Updating the Rust ecosystem will require time and cannot begin before these
    preliminaries are available.

- What other designs have been considered and what is the rationale for not choosing them?

    `stride_of` could return `0` for ZSTs.
    This was not chosen, as the choice of having ZSTs return `align_of` for `stride_of`
    enables greater correctness, as while the naive allocation and pointer arithmetic
    would still be inefficient, as it would both request memory from the system allocator
    and not pack 2+aligned ZSTs, it would not fail as the current naive solution does,
    while still enabling such optimisations using `store_size_of`.

- What is the impact of not doing this?

    Complete interoperability between Rust and Swift would be impossible without this
    RFC or a similar RFC.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?

    [Swift](swift.org) provides `store_size_of` and `stride_of` available as
    [`MemoryLayout`](https://swiftdoc.org/v5.1/type/memorylayout/)`.size` and `.stride` respectively.
    Swift's default layout algorithm also does distinguish between size and stride.
    However, Swift provides less utility to pointers, and does not require that objects
    remain in a specific location in memory unless explicitly requested, such as with a
    pointer-forming function call, which only lasts to the end of the called function.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

    Should `store_size_of<char>()` return 4 or 3?
    Swift uses 4, but it is only well defined to 21 bits.

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

    None.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

    Should Rust support `#[repr(Swift_5)]`?
    Support within rustc for layouts where size is distinct from stride.

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

###

- support sub-byte layout algorithms?
  - This would presumably enable generalizing over bit-packing 
    and the concept of niches in rustc.
