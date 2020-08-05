# Summary
Add the anonymous enum type. Anonymous Enums would have a relation to Enums similar to Tuples relationship to structs.

# Motivation 

This is not the first proposal of an Anonymous Enum like feature. Both anonymous enums and Union like RFCs have been proposed in the past, but none of these RFCS have been approved for implementation. In general the union like RFCs face challenges due to an explosion of edge cases revolving around lifetimes and generic code. While the enum like RFCs face criticism about the ergonomics as they fall behind those of the union RFCs. In order for one of these RFCs to make ground, either the union like RFCs need to solve the handling of generic and lifetime edge cases, or an anonymous enum proposal needs to match the ergonomics of a union. This proposal aims to address the later challenge.

Anonymous Enums at their core are simple, they are strongly typed, they have indexed variants, and variants are accessed through index matching. But they serve as a foundation for more advanced features such as Type and Trait matching syntactic sugar that desugars to index matching. As well as coercion to transform one anonymous enum into another.

With the resiliency of indexed variants and the ergonomics of type matching you would be able to write error handling code like this.
```rust
// Returns enum(io::Error, ParseIntError)
fn int_from_file() -> Result<i32, enum impl Error> {
    let mut file = File::open(file_name)?; // Can produce io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // Can produce io::Error
    let output = i32::from_str_radix(&contents, 10)?; // Can produce ParseIntError
    Ok(output)
}

// Type and Trait matching
match int_from_file() {
    Ok(i) => ... 
    Err(io : IoError) => ...;           // Type Match
    Err(parse: ParseIntError) => ...;   // Type Match
    Err(e : impl Error) => ...;         // Trait Match
}
```

It is also possible to merge multiple types that implements the same trait into a new one.
```rust
let some_data = vec![
    vec![1, 2, 3],
    vec![4, 5, 6],
];
let new_data: Vec<usize> = some_data
    .iter()
    .flat_map(|line| -> enum impl Iterator {
        // This will create an iterator from the union of two different itertor types
        if some_condition {
            line.iter() as enum
        } else {
            line.iter().rev() as enum
        }
    })
    .collect();

// if some_condition is true,
// new_data == vec![1, 2, 3, 4, 5, 6]
// otherwise
// new_data == vec![3, 2, 1, 6, 5, 4]
```

# Explanation
## Index level syntax
Anonymous Enums are enums without a type name or variant names. They are declared using the `enum` keyword followed by the variant types, separated by commas. Dot syntax similar to tuples are used to specify a variant. If the type or position can be inferred they can be omitted.
```rust
// Declare an anonymous enum
let mut x : enum(i32, bool);
// a full path example of coercing a type into an enum
x  = false as enum(i32, bool).1;
// Type can be inferred
x = true as enum.1;
// So can position when the value is a unambiguous top level variant type
x = 1i32 as enum

// Index matching can infer type based on the type it is matching
match x {
    enum.0(val) => assert_eq!(val, 1i32),
    enum.1(_) => unreachable!("Value was set to the first variant")
}
```
## Coercion
Anonymous Enums would make use of coercion for ergonomics, but the coercions would still be explicit. There are two types of coercions: assignment and transformation. Coercions are triggered by `as enum`. The type and position will be inferred unless specified.

For assignment coercion to occur, There must be one and only one variant with a pre-monomorphic top level type equal to the type being coerced. This coercion would serve the common case of taking a type and moving it into an anonymous enum.
```rust
// Ok, the first variant is assigned
let y = i32 as enum(i32, u32);
// Error ambiguous assignment, specify a position
let y = i32 as enum(i32, i32);
// Error no top level variant of type i32, specify a position
let z = i32 as enum(u32, enum(i32, bool));
```
A Transformation coercion occurs when converting from one anonymous enum into another. The coercion is valid if all variants of the right hand side can be coerced into a variant on the left hand side. If a variant of the right hand side is an enum not present on the left hand side, the coercion will recurse. This recursion allows for the flattening of enums. Transformation coercion would commonly be used to flatten or expand enums as they propagate upwards.
```rust
// This is an example of 2 enums that coerce into a larger enum
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)> {
    if random() {
        int_from_file() as enum // returns enum(io::Error, ParseIntError)
    } else {
        int_from_db() as enum // returns enum(sql::Error, http::Error)
    }
}

// The ? operator would also perform these coercions 
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)> {
    if random() {
        int_from_file()? // returns enum(io::Error, ParseIntError)
    } else {
        int_from_db()? // returns enum(sql::Error, http::Error)
    }
}
```
Coercions would make assignments based on the pre-monomorphic types. This means that assignments would be  consistent regardless of the concrete type of generics. This is to ensure that a value assigned in a generic context will never collapse a generic type into a concrete type or vise versa.
```rust
let t : T = ...
let mut u8_or_t : enum(u8, T);
u8_or_t = 5u8 as enum; // Always gets assigned to variant 0, regardless of T's type
u8_or_t = t as enum; // Always gets assigned to variant 1, regardless of T's type

// The same pre-monomorphic assignment logic applies to transformation
// coercion. u8 always maps to u8, and T always maps to T, regardless of T's type
let u8_or_t_or_i32 : enum(u8, T, i32) = u8_or_t as enum;
```
When performing a transformation coercion, it is possible to coerce multiple variants of the same type into one. However the resulting type will have the minimum lifetime.
```rust
let lifetimes : enum(&'a u32, &'b u32, bool);
let merged : enum(&'c u32, bool) = lifetimes as enum;

`c <= `a 
`c <= `b
```
## Type Matching
Type matching would be syntactic sugar over an index match. Types are mapped to indexes before monomorphization. This is to prevent situations where a generic type is the same as a non generic type and the branch arms would conflict. Only top level types are considered for type matching. If multiple top level variants have the same type, the branch is duplicated for each variant.

The syntax for a type match branch would be the variable name, followed by a colon, followed by the local pre-monomorphic top level type of the enum being matched.
```rust
// Type matching
let err : enum(io::Error, parse::Error, E, E) = ...
match err {
    io : IoError => Expr1;
    parse: ParseIntError => Expr2;
    e : E => Expr3;
}
```
The above type match would be desugared into the following index match syntax.
```rust
// Type matching
let err : enum(io::Error, parse::Error, E) = ...
match x {
    enum.0(io) => Expr1;
    enum.1(parse) => Expr2;
    enum.2(e) => Expr3;
    enum.3(e) => Expr3;
}
```
Because assignment coercion and type matching both operate on pre-monomorphic types, assigning a value to err with a type of E will result in Expr3 being run, regardless if E happens to be an io::Error or ParseIntError. The reason this is important is that a generic variant wouldn't be cast on a concrete type, and a concrete type wouldn't be cast to a variant. This is especially important when the generic type and the concrete type are defined with different lifetimes.

## Trait Matching
Trait matching would be syntactic sugar over an index match. Like coercion and type matching, traits are mapped to indexes before monomorphization. This means a generic type will need to be bound to the trait in order for the match to occur.

The syntax for a trait match branch would be the variable name, followed by a colon, followed by impl Trait.
```rust
// Type matching
let err : enum(io::Error, parse::Error, E) = ...
match err {
    err : impl Error => Expr1
    e : E => Expr2; // E is not guaranteed to implement Error
}
```
The above type match would be desugared into the following index match syntax.
```rust
// Type matching
let err : enum(io::Error, parse::Error, E) = ...
match x {
    enum.0(err) => Expr1;
    enum.1(err) => Expr1;
    enum.2(e) => Expr2;
}
```
## Mix and Matching
Since type matching and trait matching are just syntactic sugar of index matching, all 3 can be used in the same match expression. This serves the case when an error is returned, and specific errors need to be handled, but other errors can simply be logged or otherwise generically handled.
```rust
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)>;
match failable(){
    Ok(_) => Expr0,
    Err(io : io::Error) => Expr1,    // Type match
    Err(http : http::Error) => Expr2,// Type match
    Err(e : impl Error) => Expr3,    // Trait match
}
```
This would desugar into the following index match
```rust
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)>;
match failable(){
    Ok(_) => Expr0,
    Err(enum.1(io)) => Expr1,    
    Err(enum.3(http)) => Expr2,  
    Err(enum.2(e)) => Expr3, 
    Err(enum.4(e)) => Expr3, 
}
```
## Enum Impl Trait
In many instances enums would be used in the return path, and new types will be added as Errors propagate up the stack. Additionally a change in a lower called function would require a change in signature in all functions that propagate the enum. To handle these cases an extension and addition to the impl trait feature would be added. The enum impl trait return type would allow the compiler to return a normalized enum consisting of all assigned types. 

All enum impl traits would also be non-exhaustive. This would require the user to perform a trait match on the enum to handle a later change in signature. 
```rust
// This function header 
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)>;
// Would become this
fn failable() -> Result<_, enum impl Error>;
```

When using `Result<_, enum impl Error>` as a better equivalent of typed exception, it would be useful to be able to:
- modify any leaf of the call-chain without having to modify all the caller (like unchecked exception)
- match exaustively on all concrete types

In order to satisfy both of those needs, it should be possible to coherce an `enum impl Trait` into `enum (T, U, V)` with `T`, `U` and `V` the complete set of types contained in the enum. Doing so would generally be done on the return type of externally visible `pub` functions, and in `match` expression.

Since the concrete types of the variant would not appear in the type declaration (`enum impl Error`) or only partially (`enum (io::Error, ..) impl Error`), being able to transform an `enum impl Trait` into `enum (T, U, V)` requires a global analysis of the program. In order to limit the size of such analysis, such conversion cannot be done across crate boundary.

```rust
fn failable() -> Result<_, enum(io::Error, sql::Error, http::Error, ParseIntError)>;
fn call_failable() -> Result<_, enum impl Error> {
    match failable() {
        Err(parse_error: ParseIntError) => {
            // this error is not propagated
        },
        // non exaustive match, new error can be added to `failable()` without having to modify `call_failable()`
        Err(error: Error) => {
            return Err(error) // propagate the error
        },
        Ok(ok) => ...,
    };
    
    // ...
}
fn usage() {
    match call_failable() {
        // exaustive match, no `Err(_: Error)` case
        Err(io: io::Error) => ...,
        Err(sql: sql::Error) => ...,
        Err(http: http::Error) => ...,
        Ok(ok) => ...,
    }
}
```

- Removing `sql::Error` from the anonymous enum returned by `failable` would requires to update `usage()` but not `call_failable()` since the later only forwards the error returned by `failable()` in an opaque way.
- Likewise adding a new error in the anonymous enum returned by `failable()` only requires to update `usage()` and not `call_failable()`.
- Removing `ParseIntError` from `failable()` would requires to update `call_failable()` since there is a partial match on this variant.

## Putting it all together
The proposal would result in error handling code that uses a pattern similar to those used in Java while codifying practices that are already used in rust. Rather than creating and mapping custom enums, anonymous enums can be used quickly and conveniently. This also creates a match syntax, that would be familiar to those why have used exceptions. The enum impl trait would ensure that if new Errors are created by the function or any of it's dependencies, callers would not have to change 
```rust
// Returns enum(io::Error, ParseIntError)
fn int_from_file() -> Result<i32, enum impl Error> {
    let mut file = File::open(file_name)?; // Can produce io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // Can produce io::Error
    let output = i32::from_str_radix(&contents, 10)?; // Can produce ParseIntError
    Ok(output)
}

// Type and Trait matching
match int_from_file() {
    Ok(i) => ...                        // Happy Match
    Err(io : IoError) => ...;           // Specific Error Match
    Err(parse: ParseIntError) => ...;   // Specific Error Match
    Err(e : impl Error) => ...;         // Catch-all Error Match
}
```
## Prior Art
The following were explicitly brought up during the work shop of the above proposal.
- **C++ Variant:** The [std::variant](https://en.cppreference.com/w/cpp/utility/variant) is a similar implementation that uses both type and indexed based matching. Like anonymous enums, the variant is a discriminated union
- **TypeScript Union:** The [Union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types) is conceptually similar, but in implementation is quite different. The union is essentially a reference to a heap allocated object that is downcast into a type.
- **Anonymous Variant RFC**: This [RFC](https://github.com/eaglgenes101/rfcs/blob/2c8e89811a64b139a62d199c5f8e5bd3e852102c/text/0000-anonymous-variants.md) by eaglegenes101 also proposed an indexed based discriminant union type for rust.
___
## Alternatives

### Union Type
The union type would exhibit behavior similar to that of TypeScript's union while having an implementation similar to a discriminated union. The union would be implicitly flattened, order independent, and equivalent to all other unions with the same variants, and they would exclusively use type matching post-monomorphization. This approach has challenges, a complicated generic match can create ambiguous behavior to the user. Multiple variants with the same type but different lifetimes would be collapsed. 

### Deref type matching
Currently the most convenient way to propagate multiple errors is by creating a boxed trait object. This does require an allocation and downcasts to perform specific error handling based on type. But type matching could instead be sugar for downcast_ref. 
