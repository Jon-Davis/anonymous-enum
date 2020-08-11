# Summary
Add the anonymous enum type. Anonymous Enums would have a relation to Enums similar to Tuples relationship to structs.

# Motivation 

There are three primary use cases for Anonymous Enums

1. Anonymous Enums can serve as enums for Anonymous Types. 

```rust
fn return_iter(some_condition: bool) -> impl Iterator<Item=u64> {
  if some_condition {
        (0..100).map(|x| x * 2).into()
    } else {
        (0..100).rev().into()
    }
}
```

2. Anonymous Enums can serve as a simple solution for functions that may return multiple types of Errors. Anonymous Enums don't aim to replace crate level Errors, but for simple user defined functions, they can reduce the friction of writing functions that perform a variety of failable tasks.

```rust
// Function can return multiple types of Errors
fn int_from_file() -> Result<i32, enum(io::Error, ParseIntError)> {
    let mut file = File::open(file_name)?;    // Can produce io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;      // Can produce io::Error
    i32::from_str_radix(&contents, 10).into() // Can produce ParseIntError
}

// Type matching 
match int_from_file() {
    Ok(_) => ...,
    Err(io : io::Error) => ...,
    Err(pi : ParseIntError) => ...,
}
```

3. Anonymous Enums can serve as a convenient Either of any arity.

```rust
let rs : enum(PostgresReader<Row>, MySqlReader<Row>) = read_from_some_database();

// Index matching 
match rs {
    enum.0(pg) => ..., // Similar to Left
    enum.1(ms) => ..., // Similar to Right
}

// Anonymous enums derive traits like Iterator or Stream
for row in rs {
    ...
}
```

# Explanation
## Anonymous Enum Type
The Anonymous enum type is defined using the keyword `enum` followed by an open and closed parenthesis. Inside the parenthesis will be a list of types separated by commas detailing the variants of the enum. The ordering of variants in the enums is important as the core matching and assignments will be index based.
```rust
let floating_point : enum(f32, f64) = ...;
let error : enum(io::Error, ParseIntError) = ...;
```

## Coercing into an Enum
Rust currently provides no implicit type conversion between primitive types, and anonymous enums will be consistent with this. Types can be coerced into an Anonymous Enum using the `as` keyword followed by the `enum` keyword. The enums variants are specified in parenthesis followed by a dot and a zero indexed number specifying which variant to assign. If the type of enum can be inferred then it can be omitted. 
```rust
let mut floating_point_or_t : enum(f32, T);
floating_point_or_t = 5f32 as enum(f32, T).0;
floating_point_or_t = t as enum.1; 
```

## From Variant
The From trait will automatically be implemented on anonymous enums for all of their top level variants where there is no overlap between any of the variants. The From implementation servers the following functions:
* Reduce the verbosity of assigning a type to a variant when the mapping is obvious.
* Add support for the `?` operator allowing functions to propagate errors into an enum.
```rust
// From will be implemented for all types
let mut floating_point : enum(f32, f64);
floating_point = 32f32.into();
floating_point = 64f64.into();
// From will not be implemented as T overlaps with f32
let mut no_from : enum(f32, T);
// From will not be implemented as there is a duplicate type
let mut no_from : enum(f32, f32);
// From will be implemented for all types
let mut floating_point : enum(f32, Vec<T>);
floating_point = 32f32.into();
floating_point = vec!(t).into();
// From will be implemented only for f32
let mut floating_point : enum(f32, Vec<T>, Vec<bool>);
floating_point = 32f32.into();
```
 
## From Enum
The From trait will automatically be implemented on anonymous enums to support conversion from one anonymous enum to another. If the destination enum implements From for all variants of the source enum, then the destination enum implements From for the source enum. The two enums do not need to have the same arity. The From implementation servers the following functions:
*  Expanding an enum. For example from `enum(io::Error, ParseIntError)` to `enum(io::Error, ParseIntError, sql::Error)`
*  Reordering an enum. For example from `enum(io::Error, ParseIntError)` to `enum(ParseIntError, io::Error)`
*  Reducing duplicates in an enum, usually before a type match. For example from `enum(u8, u8, f32)` to `enum(u8, f32)`
*  Flattening an enum. For example from `enum(u8, enum(u16, u32))` to `enum(u8, u16, u32)`
```rust
let floating_points : enum(f32, f64) = 32f32.into();
let unsigned : enum(u32, u64) = 32u32.into();
let five_f64s = 64f64 as enum(f64, f64, f64, f64, f64).0;

let mut numbers : enum(f32, f64, u32, u64);
numbers = floating_points.into();
numbers = unsigned.into();
numbers = five_f64s.into();
```

## TryFrom Enum
The TryFrom trait will automatically be implemented on anonymous enums to support conversion from one anonymous enum to another. If the destination enum implements From for some of the variants of the source enum, then the destination enum implements TryFrom for the source enum. The two enums do not need to have the same arity. The TryFrom implementation servers the following functions:
* Reducing an enum, for example from a `enum(u32, u64, f32, f64)` into a `enum(f32, f64)`
* Checking if two enums intersect, for example `enum(bool, u8, f32)` into `enum(bool, f32, String)`
```rust
let mut numbers : enum(f32, f64, u32, u64) = 5f32;
let floating_points : enum(f32, f64) = numbers.try_into()?;
let unsigned : enum(u32, u64) = numbers.try_into()?;
```

## Index Matching
The core matching syntax of anonymous enums would be indexed based. Each branch will begin with the keyword `enum` followed by the index the branch arm matches with. Then a variable name to bind the result to is specified in parenthesis. Index Matching has the following functions:
* Index matching serves as the foundation that other forms of matching such as type matching desugar into.
* Index matching can disambiguate enums with duplicate types, such as `enum(u8, u8)`
* Index matching can sometimes be cleaner than Type matching, such as in heavily nested generic types.
```rust
match int_from_file().unwrap_err() {
    enum.0(io) => ..., // io::Error
    enum.1(pi) => ..., // ParseIntError
}
```

## Type Matching
Type matching is syntactic sugar over Index Matching. Local pre-monomorphic types are used for types and must unambiguously map to only one variant of an enum. A type match begins with a variable name, followed by a colon, and a local pre-monomorphic type. 
* Type matching does not require the user to know the exact ordering of the enum when all they care about is the type
* Type matching would be familiar to users of other languages like java and python as it provides similar error handling code to a Try block. Matching Errors on type, similar to how Exceptions are caught.
* Type matching can be cleaner to read than index matching
```rust
// A "Try Catch" like type matching 
match int_from_file() {
    Ok(_) => ...,
    Err(io : io::Error) => ...,     // enum.0(io)
    Err(pi : ParseIntError) => ..., // enum.1(pi)
}

// A Generic type matching
let e : enum(f32, T) = ...;
match e {
    f : f32 => ..., // enum.0(f) => ..., Regardless of T
    t : T => ...,   // enum.1(t) => ..., Regardless of T
}
```

## Product Safe Traits
Anonymous enums would introduce "Product Safe Traits". This would be an analogy to object safe traits. These are traits that would allow anonymous enums to implicitly derive any trait shared by all variants. Product Safe Traits  will has the following functions:
* For error handling if all variants implement Error, then the enum can be used as an Error type for logging
* This enables `enum impl Trait`
```rust
// A "Try Catch" like type matching 
match int_from_file() {
    Ok(_) => ...,
    Err(io : io::Error) => ..., // Match on specific error
    Err(e) => ...,              // Match on any error, e : Error
}

// Deriving traits like Iterator or Stream allows the user to use the traits regardless of the concrete type
let rs : enum(PostgresReader<Row>, MySqlReader<Row>) = read_from_some_database();
for row in rs {
    ...
}
```

## Enum Impl Trait
Enum Impl Trait would be analogous to the `impl Trait` feature already in rust. The function would return one enum that contains the different types the function could return as variants. Even though the return type is an enum, the variants are opaque, and as a result the enum can not be matched on using index or type matching. Enum Impl Trait will has the following functions:
* Allow functions to return multiple anonymous types without needing allocation
* Allow a function to simply return an opaque Error, with the ability to add more causes without modifying the header.
```rust
// Example in a normal function
fn return_iter(some_condition: bool) -> enum impl Iterator<Item=u64> {
  if some_condition {
        (0..100).map(|x| x * 2).into()
    } else {
        (0..100).rev().into()
    }
}

// Example in a flat_map
vec!((0..100), (100..200))
    .iter()
    .flat_map(|line| -> enum impl Iterator {
        if some_condition {
            line.map(|x| x * 2).into()
        } else {
            line.rev().into()
        }
    });

// Example returning some Error type
fn int_from_file() -> Result<i32, enum impl Error> { ... }
```
# Teaching Anonymous Enums

## For Error handling 
Many languages use Exceptions as their primary means of Error handling. Especially those programers are first introduced to in University like Java and Python. In these languages Exceptions are thrown from their source up the call stack until a try-catch or try-except block handles them. There the exceptions are matched based on type, or a generic catch all exception can be used.
```java
int openFileParseInt() throws FileNotFoundException, ParseException, ... { ... }

try {
    int i = openFileParseInt();     // Success
    ...
} catch (FileNotFoundException f) { // File Error
    ...
} catch (ParseException p) {        // Parse Error
    ...
} catch (Exception e) {             // Generic Error
    ...
}
```
The above java code is an example of try-catch. The below rust code is an example using anonymous enums.
```rust
fn int_from_file() -> Result<i32, enum(io::Error, ParseIntError, ...)> { ... }

// Type matching 
match int_from_file() {
    Ok(_) => ...,                   // Success
    Err(io : io::Error) => ...,     // File Error
    Err(pi : ParseIntError) => ..., // Parse Error
    Err(e) => ...,                  // Generic Error
}
```
Two different things are occurring as exceptions are different from enums, however the rust code would be familiar to people who have used exceptions in other languages. If a person is able to understand that an enum is used to handle multiple possible error types, it would not be a far leap to learn that enums can be used for other purposes outside of Error handling. 

## Outside of Error handling
Anonymous Enums share a similar relationship to enums that tuples share with structs. Using the dot syntax for both index matching and assignment coercion is no coincidence. Teaching that anonymous enums are just enums with some special trait derive logic, and tuple like assignments would be another method to teaching the feature.


# Prior Art
The following were explicitly brought up during the work shop of the above proposal.
- **C++ Variant:** The [std::variant](https://en.cppreference.com/w/cpp/utility/variant) is a similar implementation that uses both type and indexed based matching. Like anonymous enums, the variant is a discriminated union
- **TypeScript Union:** The [Union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types) is conceptually similar, but in implementation is quite different. The union is essentially a reference to a heap allocated object that is downcast into a type.
- **Anonymous Variant RFC**: This [RFC](https://github.com/eaglgenes101/rfcs/blob/2c8e89811a64b139a62d199c5f8e5bd3e852102c/text/0000-anonymous-variants.md) by eaglegenes101 also proposed an indexed based discriminant union type for rust.
___
# Alternatives

## Union Type
The union type would exhibit behavior similar to that of TypeScript's union while having an implementation similar to a discriminated union. The union would be implicitly flattened, order independent, and equivalent to all other unions with the same variants, and they would exclusively use type matching post-monomorphization. This approach has challenges, a complicated generic match can create ambiguous behavior to the user. Multiple variants with the same type but different lifetimes would be collapsed. 

## Status Quo 
Most of the features anonymous enums offer can already be achieved in rust. Named enums are verbose and require a large amount of boiler plate or macros, but they are able to achieve most of what anonymous enums could do. What can't be handled by named enums could be achieved through trait objects, such as providing an interface over anonymous types. Although trait objects would require an allocation. Anonymous enums do not extend the language, allowing it to do things previously impossible, they aim to simply reduce friction for certain use cases.
