- Feature Name: Allow Macros in Attribute Values
- Start Date: 2018-01-10
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow macros and meta-variables wherever an attribute expects a literal. For example:

```rust
#[path = concat!(env!("OUT_DIR"), "/hello.rs")]
mod foo;
```

```rust
macro_rules! linked {
  ($name:ident, $link_name:expr) => {
    #[link_section($link_name)]
    pub static $name: usize = 0;
  }
}

linked!(X, ".section_for_X");
```

# Motivation
[motivation]: #motivation

When users learn to define macros or are introduced to string literal macros like `concat!` and `include_str!`, they sometimes attempt to combine these with attributes. However, this natural progression currently leads to parser errors ([rust-lang/rust#18849](https://github.com/rust-lang/rust/issues/18849)) or ICEs ([rust-lang/rust#42164](https://github.com/rust-lang/rust/issues/42164)).

One desired use case was `#[doc(include_str!("path/to/doc.txt"))]` to allow ergonomically keeping documentation outside of Rust source files. This was eventually emulated by the accepted [RFC 1990](https://github.com/rust-lang/rfcs/pull/1990), indicating that macros in attributes could be used to solve problems at least important enough to go through the RFC process.

Other use cases revolve around the attributes `#[path]` for modules, and `#[export_name]` and `#[link_section]` for static and function declarations. Currently the only way to programmatically generate declarations using these attributes is code generation e.g. via `build.rs`. This seems like an unnecessary hurdle for what feels like a natural extension of how macros are used.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Attributes As Macros
[guide-attributes]: #guide-attributes

Attributes are annotations of the form `#[name(args...)]` that are included just above various item declarations, or of the form `#![name(args...)]` at the top of a module.

An attribute is either provided by a crate as part of a procedural macro, or is defined by the language. In both cases, the attribute is treated similarly to a macro, in that it is passed its arguments in the form of a token tree and it is the responsibility of the attribute definition to parse and interpret these as intended.

As an example, consider implementing a custom `#[derive(HelloWorld)]` procedural macro as discused in [the book](https://doc.rust-lang.org/book/first-edition/procedural-macros.html). In the section on [custom attributes](https://doc.rust-lang.org/beta/book/first-edition/procedural-macros.html#custom-attributes), we see how to extend Rust to allow a custom attribute `HelloWorldName`, used as follows:

```rust
#[derive(HelloWorld)]
#[HelloWorldName = "the best Pancakes"]
struct Pancakes;
```

If you were using the [`syn`](https://github.com/dtolnay/syn) crate to parse the Rust syntax tree passed to your procedural macro, that custom attribute would be parsed as an instance of [`syn::Attribute`](https://docs.rs/syn/0.12/syn/struct.Attribute.html) which could be further parsed into an instance of [`syn::Meta`](https://docs.rs/syn/0.12/syn/enum.Meta.html). The final result would look something like this:

```rust
use syn::*;

Meta::NameValue(
    MetaNameValue(
        ident: Ident("HelloWorldName"),
        lit: Lit::Str(LitStr("the best Pancakes")),
        ...
    )
)
```

As the implementor of the `HelloWorld` custom derive macro, you would need to check that instances of your custom attribute `HelloWorldName` were used correctly by checking the structures produced above. For instance, you may only allow the `#[name = "literal"]` attribute format. Looking at the documentation for `syn::Meta`, this corresponds to expecting a `NameValue` item after being parsed. Similarly, we can use this parsing result to extract configuration data from custom attributes. In this case, `lit` above corresponds to the right hand side of the equal sign in our custom attribute (`= "the best Pancakes"`).

## Macros in Attributes
[guide-complication]: #guide-complication

There are two uncommon, but not unexpected complications to the above story of how to parse attributes. The first is that someone using your custom attribute might pass a macro invocation instead of a literal:

```rust
#[derive(HelloWorld)]
#[HelloWorldName = env!("HELLO_WORLD_NAME")]
struct Pancakes;
```

The second is that someone might include your custom attribute inside another macro's definition, and pass along a literal or token stream using a metavariable:

```rust
macro_rules! my_hello_world {
    ($name:expr) => {
        #[derive(HelloWorld)]
        #[HelloWorldName = $name]
        struct Pancakes;
    }
}
```

In both cases, the parsing results look slightly different to those we saw above. In order to handle these complications, we need to make use of _expansions_.

TODO: Flesh out what a macro/metavariable expansion API looks like in a `syn`/`quote` world, or find an existing framework to use as an example.

## Built-In Attributes
[guide-builtin]: #guide-builtin

Some attributes, like `#[path]` and `#[doc]`, are defined as part of the Rust language and are implemented by the compiler. However, they work very similarly to the process outlined above, and so you can use macro invocations and macro variables in these yourself!

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
