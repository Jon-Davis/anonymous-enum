# Anonymous Enums
Anonymous enums would be a new primitive type added to the rust programing language. Work shopped in this [internals.rust-lang.org](https://internals.rust-lang.org/t/ideas-around-anonymous-enum-types/12627) thread.

## Core Proposal

### Strongly Typed
Anonymous Enums are strongly typed. They will not have any implicit flattening or order independence.
- `enum(A, B) != enum(B,A)` 
-  `enum(A, A) != A`
___
### Matching
The core matching would be index based, and use a syntax familiar with tuples.
```rust
let anon_enum3 : enum(A, (B, C)) = ...;
match anon_enum3 {
    0(a) => ...,
    1.0(b) => ...,
    1.1(c) => ..., 
}
```
Type matching would be syntactic sugar of indexed matching and would match on the local types before monomorphization.
```rust
let anon_enum : enum(u8, T) = ...;
match anon_enum { // match anon_enum {
   a  u8 => ...;  //     0(a) => ...;
   b: T => ...;   //     1(b) => ...;
}                 // }
``` 
___
### Assignments
- Assignments would be made based on the local types before monomorphization. 
- Only a top level Type on the lhs can be assigned. 
- Assignments must be unambiguous. (No two top level variants of the same type)
```rust
// Example 1: Simple Case (a : A)
let _: enum(u8, u16) = 0u8; // Ok assigned to 0
let _: enum(u8, u8) = 0u8;  // Err ambiguous assignment 

// Example 2: Top Level assignment 
let _: enum(u8, enum(u8, u16)) = 1u8;                   // Ok assigned to 0
let _: enum(u8, enum(u8, u16)) = 1u8 as enum(u8, u16);  // Ok assigned to 1.0

// Example 3: Generic (t : T)
let _: enum(u8, T) = 0u8; // Ok assigned to 0 regardless of T
let _: enum(u8, T) = t;   // Ok assigned to 1 regardless of T
```
___
### Coercion and Normalization 
Anonymous enums can be coerced into different anonymous enums so long as every variant on the top level of the rhs can be assigned to the lhs. 

If a variant of the rhs is an anonymous enum that is not a top level variant of the lhs, the coercion will recurse on the variant.

Like Assignments, The coercion is done on local types before monomorphization.

Expected behavior:
```rust
// Basic examples
enum(u8, u16, u32) <- enum(u16, u32, u8);       // Reordering
enum(u8, u16, u32) <- enum(u8, enum(u16, u32)); // Flattening
enum(u8, u16) <- enum(u8, u8, u16);             // Collapsing
enum(u16, u8) <- enum(u8, enum(u8, u16));       // All 3

// Generic example (0: T, 1: u8) - Regardless of T 
enum(T, u8) <- enum(u8, (u8, T)) 
```
___
### Generics
Anonymous enums determine the variant of an assignment or type match before monomorphization. This means
- Variant assignments are consistent across generic implementations
- Generics will never be collapsed into a different variant based on their actual type

```rust
// This will always assign t to the second variant
let anon_enum : enum(u8, T) = t as T;

// This will always run the second branch if anon_enum was T
match anon_enum { // match anon_enum {
   a  u8 => ...;  //     0(a) => ...;
   b: T => ...;   //     1(b) => ...;
}
```
___
### Type Matching
Anonymous enums have type matching as syntactic sugar to index matching. The logic used to describe the mapping of one anonymous enum coercing into another is the same logic used to describe the mapping of anonymous enum variants and a type match expression branches. 
```rust
// Consider the following valid coercion
enum(u16, u8) <- enum(u8, enum(u8, u16));

// Since the coercion is valid, so too is this match
let matchable : enum(u8, enum(u8, u16));

// The mapping is many to one, so branches may match on multiple variants.
// The u8 branch will match on either of the u8 variants.
match matchable {   // match matchable
    s: u16 => ..,   //     1.1(s) => ...
    b: u8 => ...,   //     0(b) | 1.0(b) => ...
}                   // }
```
___

### Traits
Anonymous enums will implicitly derive traits if all variants implement a common trait. To achieve this an analogue to object safe traits will be introduced: "product safe traits". These will be traits that can be implemented implicitly on an anonymous enum if all variants implement the trait. Traits related to error handling such as: Debug, Display, and Error are specifically important for anonymous Enums, as a primary use case will be propagating errors.
___

## Prior Art
The following were explicitly brought up during the work shop of the above proposal.
- **C++ Variant:** The [std::variant](https://en.cppreference.com/w/cpp/utility/variant) is a similar implementation that uses both type and indexed based matching. Like anonymous enums, the variant is a discriminated union
- **TypeScript Union:** The [Union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types) is conceptually similar, but in implementation is quite different. The union is essentially a reference to a heap allocated object that is downcast into a type.
- **Anonymous Variant RFC**: This [RFC](https://github.com/eaglgenes101/rfcs/blob/2c8e89811a64b139a62d199c5f8e5bd3e852102c/text/0000-anonymous-variants.md) by eaglegenes101 also proposed an indexed based discriminant union type for rust.
___
## Alternatives

### Union Type
The union type would exhibit behavior similar to that of TypeScript's union while having an implementation similar to a discriminated union. The union would be implicitly flattened, order independent, and equivalent to all other unions with the same variants, and they would exclusively use type matching post-monomorphization. This approach has challenges, a complicated generic match can create ambiguous behavior to the user. Multiple variants with the same type but different lifetimes would be collapsed. 
___
## Unresolved Questions
- Should coercion into anonymous enums be implicit? Should coercion between anonymous enums be implicit? Should a keyword become or as be used?
## Additional Proposals
The following proposals were discussed during the work shop and create additional functionality. They may be apart of an initial implementation, but are capable of being added later. In order of discussion:

### Impl Trait 
Since anonymous enums implement all product safe traits, an extension to the impl Trait functionality could be added to support returning anonymous enums as interfaces. 
```rust
// Returns enum(slice::Iter, Rev<slice::Iter>)
fn create_iterator<T>(vec: &Vec<T>, direction: Direction) -> impl Iterator<Item=T> {
    match direction {
        Direction::Forward => vec.iter(),
        Direction::Backward => vec.iter().rev(),
    }
}
```
___
### Variadic Enums  
Variadic enums would allow the declaration of anonymous enums with variable abstract concrete variants. They would likely be accompanied by Trait Matching, which would match on all top level pre-monomorphic types guaranteed to have that trait, recursing on anonymous enum variants.
```rust
// Returns enum(slice::Iter, Rev<slice::Iter>)
fn create_iterator<T>(vec: &Vec<T>, direction: Direction) -> enum(...Iterator<Item=T>) {
    match direction {
        Direction::Forward => vec.iter(),
        Direction::Backward => vec.iter().rev(),
    }
}

// a : (u8, String, i32)
let a : (...Numeric, ...Display);
a = 5u8;
a = String::from("Hello World")
a = -5i32;
```
___
### Variant Enums
Variant enums would be like a tuple struct, but for anonymous enums.
```rust
enum Variant(f32, f64);
```