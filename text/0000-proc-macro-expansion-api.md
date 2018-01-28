- Feature Name: Macro Expansion API for Proc Macros
- Start Date: 2018-01-26
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add an API for procedural macros to expand macro calls in token streams. This will allow proc macros to handle unexpanded macro calls that are passed as inputs, as well as allow proc macros to access the results of macro calls that they construct themselves.

# Motivation
[motivation]: #motivation

There are a few places where proc macros may encounter unexpanded macros in their input even after [rust/pull/41029](https://github.com/rust-lang/rust/pull/41029) is merged:

* In attribute and procedural macros:

```rust
#[my_attr_macro(x = a_macro_call!(...))]
//                  ^^^^^^^^^^^^^^^^^^
// This call isn't expanded before being passed to `my_attr_macro`, and can't be
// since attr macros are passed raw token streams by design.
struct X {...}
```

```rust
my_proc_macro!(concat!("hello", "world"));
//             ^^^^^^^^^^^^^^^^^^^^^^^^^
// This call isn't expanded before being passed to `my_proc_macro`, and can't be
// since proc macros are passed raw token streams by design.
```

* In proc macros called with metavariables or token streams:

```rust
macro_rules! m {
    ($($x:tt)*) => {
        my_proc_macro!($($x)*);
    },
}

m!(concat!("a", "b", "c"));
// ^^^^^^^^^^^^^^^^^^^^^^
// This call isn't expanded before being passed to `my_proc_macro`, and can't be
// because `m!` is declared to take a token tree, not a parsed expression that we know
// how to expand.
```

In these situations, proc macros need to either re-call the input macro call as part of their token output, or simply reject the input. If the proc macro needs to inspect the result of the macro call (for instance, to check or edit it, or to re-export a hygienic symbol defined in it), the author is currently unable to do so. This implies an additional place where a proc macro might encounter an unexpanded macro call, by _constructing_ it:

* In a proc macro definition:

```rust
#[proc_macro]
fn my_proc_macro(tokens: TokenStream) -> TokenStream {
    let token_args = extract_from(tokens);
    
    // These arguments are a token stream, but they will be passed to `another_macro!`
    // after being parsed as whatever `another_macro!` expects.
    //                                                  vvvvvvvvvv
    let other_tokens = some_other_crate::another_macro!(token_args);
    //                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // This call gets expanded into whatever `another_macro` expects to be expanded
    // as. There is currently no way to get the resulting tokens without requiring the
    // macro result to compile in the same crate as `my_proc_macro`.
    ...
}
```

Giving proc macro authors the ability to handle these situations will allow proc macros to 'just work' in more contexts, and without surprising users who expect macro calls to interact well with _other_ calls. Additionally, supporting the 'proc macro definition' use case above allows proc macro authors to use other crate macros without demanding that they be proc macros in turn.

As a side note, allowing macro calls in built-in attributes would solve a few outstanding issues (see [rust-lang/rust#18849](https://github.com/rust-lang/rust/issues/18849) for an example). 

An older motivation to allow macro calls in attributes was to get `#[doc(include_str!("path/to/doc.txt"))]` working, in order to provide an ergonomic way to keep documentation outside of Rust source files. This was eventually emulated by the accepted [RFC 1990](https://github.com/rust-lang/rfcs/pull/1990), indicating that macros in attributes could be used to solve problems at least important enough to go through the RFC process.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Macro Calls in Procedural Macros

When implementing procedural macros you should account for the possibility that a user might provide a macro call in their input. For example, here's a silly proc macro that evaluates to the length of the string literal passed in.:

```rust
extern crate syn;
#[macro_use]
extern crate quote;

#[proc_macro]
fn string_length(tokens: TokenStream) -> TokenStream {
    let lit: syn::LitStr = syn::parse(tokens).unwrap();
    let len = str_lit.value().len();
    
    quote!(#len)
}
```

If you call `string_length!` with something obviously wrong, like `string_length!(struct X)`, you'll get a parser error when `unwrap` gets called, which is expected. But what do you think happens if you call `string_length!(stringify!(struct X))`?

It's reasonable to expect that `stringify!(struct X)` gets expanded and turned into a string literal `"struct X"`, before being passed to `string_length`. However, in order to give the most control to proc macro authors, Rust doesn't touch any of the ingoing tokens passed to a proc macro (**Note:** this doesn't strictly hold true for [proc _attribute_ macros](#macro-calls-in-attribute-macros)).

Thankfully, there's an easy solution: the proc macro API offered by the compiler has methods for constructing and expanding macro calls. The `syn` crate uses these methods to provide an alternative to `parse`, called `parse_expand`. As the name suggests, `parse_expand` parses the input token stream while expanding and parsing any encountered macro calls. Indeed, replacing `parse` with `parse_expand` in our definition of `string_length` means it will handle input like `stringify!(struct X)` exactly as expected.

As a utility, `parse_expand` uses sane expansion options for the most common case of macro calls in token stream inputs. It assumes:

* The called macro, as well as any identifiers in its arguments, is in scope at the macro call site.
* The called macro should behave as though it were expanded in the source location.

To understand what these assumptions mean, or how to expand a macro differently, check out the section on how [macro hygiene works](#spans-and-scopes) as well as the detailed [API overview](#api-overview).

## Macro Calls in Attribute Macros

Macro calls also show up in attribute macros. The situation is very similar to that of proc macros: `syn` offers `parse_meta_expand` in addition to `parse_meta`. This can be used to parse the attribute argument tokens, assuming your macro expects a normal meta-item and not some fancy custom token tree. For instance, the following behaves as expected:

```rust
#[proc_macro_attribute]
fn my_attr_macro(attr: TokenStream, body: TokenStream) -> TokenStream {
    let meta: Syn::Meta = syn::parse_meta_expand(attr).unwrap();
    ...
}
```

```rust
// Parses successfully: `my_attr_macro` behaves as though called with
// ``my_attr_macro(value = "Hello, world!")]
// struct X {...}
//                      vvvvvvvvvvvvvvvvvvvvvvvvvvvv
#[my_attr_macro(value = concat!("Hello, ", "world!"))]
struct X {...}

// Parses unsuccessfully: the normal Rust syntax for meta items expects
// a literal, not an expression.
//                      vvvvvvvvvvvvvvvvvvvvvvvvv
#[my_attr_macro(value = println!("Hello, world!"))]
struct Y {...}
```

Of course, even if your attribute macro _does_ use a fancy token syntax, you can still use `parse_expand` to handle any macro calls you encounter.

**Note:** Because the built-in attribute 'macro' `#[cfg]` is expanded and evaluated before body tokens are sent to an attribute macro, the compiler will also expand any other macros before then too for consistency. For instance, here `my_attr_macro!` will see `field: u32` instead of a call to `type_macro!`:

```rust
macro_rules! type_macro {
    () => { u32 };
}

#[my_attr_macro(...)]
struct X {
    field: type_macro!(),
}
```

## Spans and Scopes
[guide-sshm]: guide-sshm

If you're not familiar with how spans are used in token streams to track both line/column data and name resolution scopes, here is a refresher. Consider the following proc macro:

```rust
#[macro_use]
extern crate quote;

#[proc_macro]
fn my_hygienic_macro(tokens: TokenStream) -> TokenStream {
    quote! {            
        let mut x = 0;  // [Def]
        #tokens         // [Call]
        x += 1;         // [Def]
    }
}
```

Each token in a `TokenStream` has a span, and that span tracks where the token is treated as being created - you'll see why we keep on saying "treated as being created" rather than just "created'" in a little bit! In the above code sample,

* The tokens in lines marked `[Def]` have spans with scopes that indicate they should be treated as though they were defined here in the definition of `my_hygienic_macro`.
* The tokens in lines marked with `[Call]` keep their original spans and scopes, which in this case indicate they should be treated as though they were defined at the macro call site, wherever that is.

Now let's see what happens when we use `my_hygienic_macro`:

```rust
fn main() {
    my_hygienic_macro! {
        let mut x = 1;
        x += 2;
    };
    println!(x);
}
```

After the call to `my_hygienic_macro!` in `main` is expanded, `main` looks something like this:

```rust
fn main() {
    let mut x = 0; // 1. [Def]
    let mut x = 1; // 2. [Call]
    x += 2;        // 3. [Call]
    x += 1;        // 4. [Def]
    println!(x);   // 5. [Call]
}
```

As you can see, the macro expansion has interleaved tokens provided by the caller (marked with `[Call]`) and tokens provided by the macro definition (marked with `[Def]`). 

Different scopes (`[Call]`, `[Def]`) define different names. That means lines 1 and 2 above each define a variable named `x` but living in a different name resolution scope. This in turn means that the `x` defined in line 2 isn't shadowing the `x` defined in line 1, which is what would happen if these variables were all declared in a single scope.

Scopes are also used to _resolve_ names. In lines 3 and 5 the variable `x` is in the `[Call]` scope, and so will resolve to the variable declared in line 2. Similarly, in line 4 the variable `x` is in the `[Def]` scope, and so will resolve to the variable declared in line 1. This also means mutating a variable in one scope doesn't mutate the variables in another, or shadow them, or interfere with name resolution. This is how Rust achieves macro hygiene.

Letting `[Scope]x` be shorthand for "the variable `x` resolved in the scope `Scope`" we can trace the execution of `main`, seeing how scopes separate two otherwise indistinguishable variables with the same name:

```rust
fn main() {
    let mut x = 0; // [Def]x = 0;
    let mut x = 1; // [Call]x = 1;
    x += 2;        // [Call]x = 3;
    x += 1;        // [Def]x = 2;
    println!(x);   // Prints [Call]x;
}
```

This doesn't just stop at variable names. The above principles apply to mods, structs, trait definition, trait method calls, macros - anything with a name which needs to be looked up.

Importantly, macro hygiene is _optional_: since we can manipulate the spans on tokens, we can change how a variable is resolved. For example:

```rust
extern crate proc_macro;
#[macro_use]
extern crate quote;

use proc_macro::Span;

#[proc_macro]
fn my_unhygienic_macro(tokens: TokenStream) -> TokenStream {
    let hygienic = quote_spanned! { Span::def_site(),
        let mut x = 0; // [Def]
    };
    let unhygienic = quote_spanned! { Span::call_site(),
        x += 1;
    };
    quote! {
        #hygienic   // [Def]
        #tokens     // [Call]
        #unhygienic // [Call]
    }
}
```

If we call `my_unhygienic_macro` instead of `my_hygienic_macro` in `main` as before, the result is:

```rust
fn main() {
    let mut x = 0; // 1. [Def]
    let mut x = 1; // 2. [Call], from main
    x += 2;        // 3. [Call], from main
    x += 1;        // 4. [Call], from my_unhygienic_macro
    println!(x);   // 5. [Call]
}
```

By changing the scope of the span of the tokens on line 4 (using `quote_spanned` instead of `quote`), that instance of `x` will resolve to the one defined on line 2 instead of line 1. In fact, the variable actually declared by our macro, on line 1, is never used.

This trick has a few uses, such as 'exporting' a name to the caller of the macro. If hygiene was not optional, any new functions or modules you created in a macro would only be resolvable in the same macro.

There are also some interesting [examples](https://github.com/dtolnay/syn/blob/030787c71b4cfb2764bccbbd2bf0e8d8497d46ef/examples/heapsize2/heapsize_derive/src/lib.rs#L65) of how this gets used to resolve method calls on traits declared in `[Def]`, but called with variables from `[Call]`.

## API Overview

The full API provided by `proc_macro` and used by `syn` is more flexible than suggested by the use of `parse_expand` and `parse_meta_expand` above. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[alternatives]: #alternatives

# Unresolved questions
[unresolved]: #unresolved-questions
