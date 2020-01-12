# spez

Macro to specialize on the type of an expression.

This crate implements *auto(de)ref specialization*:
A trick to do specialization in non-generic contexts on stable Rust.

For the details of this technique, see:
 - [*Autoref-based stable specialization* by David Tolnay][autoref]
 - [*Generalized Autoref-Based Specialization* by Lukas Kalbertodt][autoderef]

[autoref]: https://github.com/dtolnay/case-studies/blob/master/autoref-specialization/README.md
[autoderef]: http://lukaskalbertodt.github.io/2019/12/05/generalized-autoref-based-specialization.html

## What it can and cannot do

The auto(de)ref technique—and therefore this macro—is useless in generic
functions, as Rust resolves the specialization based on the bounds defined
on the generic context, not based on the actual type when instantiated.
(See the last example for a demonstration of this.)

In non-generic contexts, it's also mostly useless, as you probably already
know the exact type of all variables.

The only place where using this can make sense is in the implementation of
macros that need to have different behaviour depending on the type of a
value passed to it. For example, a macro that prints the `Debug` output of
a value, but falls back to a default when it doesn't implement `Debug`.

## How to use it

The basic syntax of the macro is:

```
spez! {
    for <expression>;
    match <type> { <body> }
    [match <type> { <body> }]
    [...]
}
```

The examples below show more details.

### Simple specialization

In the most simple case, you use this macro to match specific types:

```rust
let x = 0;
spez! {
    for x;
    match i32 {
        println!("x is a 32-bit integer!");
    }
    match &str {
        println!("x is a string slice!");
        assert!(false);
    }
}
```

### Return types

Values can be returned from the matches, but have to be explicitly
specified for each `match`. They do not have to be the same for every
`match`.

```rust
let x = 0;
let result = spez! {
    for x;
    match i32 -> &'static str {
        "x is a 32-bit integer!"
    }
    match &str -> i32 {
        123
    }
};
assert_eq!(result, "x is a 32-bit integer!");
```

### Generic matches

Generic matches are also possible. Generic variables can be defined
on the `match`, and a `where` clause can be added after the type.

The matches are tried in order. The first matches get priority over later
ones, even if later ones are perfect matches.

```rust
let x = 123i32;
let result = spez! {
    for x;
    match<T> T where i8: From<T> -> i32 {
        0
    }
    match<T: std::fmt::Debug> T -> i32 {
        1
    }
    match i32 -> i32 {
        2
    }
};
assert_eq!(result, 1);
```

## Consuming the input

The input (after the `for`) is consumed and made available to the `match`
bodies.

(If you don't want to consume the input, take a reference and also prepend
a `&` to the types you're matching.)

```rust
let x = Box::new(123);
let result = spez! {
    for x;
    match<T: Deref<Target = i32>> T -> i32 {
        *x
    }
    match i32 -> i32 {
        x
    }
};
assert_eq!(result, 123);
```

## Expressions as input

Not just variable names, but full expressions can be given as input.
However, if you want to refer to them from the match bodies, you need to
prepend `name =` to give the input a name.

```rust
let result = spez! {
    for 1 + 1;
    match i32 -> i32 { 0 }
    match i64 -> i32 { 1 }
};
assert_eq!(result, 0);
```

```rust
let result = spez! {
    for x = 1 + 1;
    match i32 -> i32 { x }
    match i64 -> i32 { 1 }
};
assert_eq!(result, 2);
```

## Capturing variables

Unfortunately, you can't refer to variables of the scope around the `spec! {}` macro:

```compile_fail
let a = 1;
let result = spez! {
    for x = 1;
    match i32 {
        println!("{}", a); // ERROR
    }
};
```

## In a generic function

As mentioned above, the macro is of not much use in generic context, as the
specialization is resolved based on the bounds rather than on the actual
type in the instantiation of the generic function:

```rust
fn f<T: Debug>(v: T) -> &'static str {
    spez! {
        for v;
        match i32 -> &'static str {
            ":)"
        }
        match<T: Debug> T -> &'static str {
            ":("
        }
        match<T> T -> &'static str {
            ":(("
        }
    }
}
assert_eq!(f(0i32), ":(");
```
