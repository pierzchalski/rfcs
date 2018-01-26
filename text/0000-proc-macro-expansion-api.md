- Feature Name: Macro Expansion API for Proc Macros
- Start Date: 2018-01-26
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add an API for procedural macros to expand resolvable macro definitions. This will allow proc macros to handle unexpanded macro invocations that are passed as inputs, as well as allow proc macros to access the results of invocations that they construct themselves.

# Motivation
[motivation]: #motivation

There are two areas where proc macros may encounter unexpanded macros in their input even after [rust/pull/41029](https://github.com/rust-lang/rust/pull/41029) is merged:

* In attribute macros:

```rust
#[my_attr_macro(x = a_macro_invocation!(...))]
//                  ^^^^^^^^^^^^^^^^^^^^^^^^
// This invocation isn't expanded before being passed to `my_attr_macro`, and can't be
// since attr macros are passed raw token streams by design.
struct S { ... }
```

* In proc macros called normally:

```rust
my_proc_macro!(concat!("hello", "world"));
//             ^^^^^^^^^^^^^^^^^^^^^^^^^
// This invocation isn't expanded before being passed to `my_proc_macro`, but could
// be expanded if we chose to.
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
// This invocation isn't expanded before being passed to `my_proc_macro`, and can't be
// because `m!` is declared to take a token tree, not a parsed expression that we know
// how to expand.
```

In these situations, proc macros need to either re-call the input macro invocation as part of their token output, or simply reject the input. If the proc macro needs to inspect the result of the macro invocation (for instance, to check or edit it, or to re-export and use a hygienic symbol defined in it), the author is currently unable to do so. This implies a third place a proc macro author might _construct_ an unexpanded macro invocation:

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
    // This invocation gets expanded into whatever `another_macro` expects to be expanded
    // as. There is currently no way to get the resulting tokens without requiring the
    // macro result to compile in the same crate as `my_proc_macro`.
    ...
}
```

Giving proc macro authors the ability to handle these situations will allow proc macros to 'just work' in more contexts, and without surprising users who expect macro invocations to interact well with _other_ invocations. Additionally, supporting the 'proc macro definition' use case above allows proc macro authors to use other crate macros without demanding that they be proc macros in turn.

As a side note, allowing macro invocations in built-in attributes would solve a few outstanding issues (see [rust-lang/rust#18849](https://github.com/rust-lang/rust/issues/18849) for an example). 

An older motivation to allow macro invocations in attributes was to get `#[doc(include_str!("path/to/doc.txt"))]` working, in order to provide an ergonomic way to keep documentation outside of Rust source files. This was eventually emulated by the accepted [RFC 1990](https://github.com/rust-lang/rfcs/pull/1990), indicating that macros in attributes could be used to solve problems at least important enough to go through the RFC process.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Macro Invocations in Proc Macros
[guide-invocations]: #guide-invocations

When implementing proc macros you should account for the possibility that a user might provide a macro invocation in their input. For example, here's a silly proc macro that evaluates to the length of a string:

```rust
extern crate syn;
#[macro_use]
extern crate quote;

#[proc_macro]
fn string_length(tokens: TokenStream) -> TokenStream {
    let str_lit: syn::LitStr = syn::parse(tokens).unwrap();
    let str_val = str_lit.value();
    let str_len = str_val.len();
    
    quote!(#str_len)
}
```

If you call `string_length` with something obviously wrong, like `string_length!(let x = 5;)`, you'll get a parser error when `unwrap` gets called, which makes sense. But what do you think happens if you call `string_length!(stringify!(let x = 5;))`?

It's not unreasonable to expect that `stringify!` gets expanded and turned into a string literal `"let x = 5;"`, before being passed to `string_length`. However, in order to give the most amount of control to proc macro authors,  

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

If you were using the [`syn`](https://github.com/dtolnay/syn) crate to parse the Rust syntax tree passed to your procedural macro, that custom attribute would be parsed as an instance of [`syn::Attribute`](https://docs.rs/syn/0.12/syn/struct.Attribute.html) which could be further parsed into an instance of [`syn::Meta`](https://docs.rs/syn/0.12/syn/enum.Meta.html).

Here is an example. We use the [`quote`](https://github.com/dtolnay/quote) crate to jump right into the parsing instead of going via a procedural macro, at the cost of having to use [`proc_macro2`](https://github.com/alexcrichton/proc-macro2) to hand-make our own AST.

```rust
extern crate syn;
#[macro_use] extern crate quote;
extern crate proc_macro2;

use syn::*;

fn main() {

    let tokens = quote! {
        #[HelloWorldName = "the best Pancakes"]
        struct Pancakes;
    };
    
    let struct: ItemStruct = parse2(tokens.into()).unwrap();
    let attribute = &struct.attrs[0];
    let parsed_attribute = attribute.interpret_meta().unwrap();
    let dummy_span = proc_macro2::Span::call_site();
    
    assert_eq!(
        parsed_attribute,
        Meta::NameValue(MetaNameValue {
        
            // Corresponds to the identifier in the left hand side of the equals sign in
            // #[HelloWorldName = "the best Pancakes"]
            ident: Ident::new("HelloWorldName", dummy_span),
            
            // The equals sign. Not interesting here but useful when reconstructing an AST
            // for reporting errors.
            eq_token: token::Eq::new(dummy_span),
            
            // Corresponds to the string literal in the right hand side of the equals sign.
            lit: Lit::Str(LitStr::new("the best Pancakes", dummy_span)),
            
        })
    );
}
```

As the implementor of the `HelloWorld` custom derive macro, you would need to check that instances of your custom attribute `HelloWorldName` were used correctly by checking the structures produced above. For instance, you might only allow the `#[name = "literal"]` attribute format. Looking at the documentation for `syn::Meta`, this corresponds to expecting a `NameValue` item after being parsed, as we see above.

## Macros in Attributes
[guide-complication]: #guide-complication

There is one uncommon, but not unexpected complications to the above story of how to parse attributes. Someone using your custom attribute might pass a macro invocation instead of a literal:

```rust
#[derive(HelloWorld)]
#[HelloWorldName = env!("HELLO_WORLD_NAME")]
struct Pancakes;
```

Recalling our previous [example](#guide-attributes), we see that our parsed syntax tree would have changed:

```rust
    assert_eq!(
        parsed_attribute,
        Meta::NameValue(MetaNameValue {
            // These are unchanged...
            ident: Ident::new("HelloWorldName", dummy_span),
            eq_token: token::Eq::new(dummy_span),
            
            // Corresponds to the macro invocation.
            lit: Lit::Macro(syn::Macro {
                // Specifies the name of the macro. This is hiding a lot of complexity - see `Path`
                // in `syn` for more.
                path: parse2(quote!(env).into()).unwrap(),
                
                // These two aren't very interesting, similar to eq_token above.
                bang_token: Bang::new(dummy_span),
                delimiter: MacroDelimiter::Paren(Paren(dummy_span)),
                
                // The arguments to the macro. These are represented as just a raw token stream,
                // which conveniently is what `quote!` returns.
                tts: quote!("HELLO_WORLD_NAME").into(),
            }),
        })
    );
```

Notice that our literal has been replaced with a macro invocation. In order to handle this complication, we need a way to turn that macro into whatever it will be expanded into, so that we can get the resulting literal expression and use it in the rest of our procedural macro (What if the macro doesn't expand into a string literal? Check out [raising errors](https://doc.rust-lang.org/beta/book/first-edition/procedural-macros.html#raising-errors)!). To do this, we need a new tool: the _expansion_ API.

Inside of a procedural macro implementation, the `proc_macro` crate has access to all the macro definitions that are in scope when your procedural macro is called. It keeps the details of this scope tracking out-of-sight but provides one function, `expand: TokenStream -> Result<TokenStream>`, which takes in a token stream for a macro invocation and returns a token stream for the result (notice the similarity to the procedural macro interface that your custom implementation also provides!).

## Using Expansions

Say we extracted the macro invocation part of the AST in our previous example, into a variable `mac`. Here is how we use `expand` to turn it into something more useful to us (in this case, a string literal):

```rust
extern crate proc_macro;
use proc_macro::expand;

fn main() {
    ...
    let mac = <our Lit::Macro node from above>;
    let mac_tokens = quote!(#mac).into();
    
    let mac_result = expand(mac_tokens)
            .expect("Failed to expand macro invocation!");
    
    let lit: Lit = parse2(mac_result)
            .expect("Failed to parse macro result as a literal!");
            
    match lit {
        Lit::Str(lit) => {
            // Success! `lit` is a `LitStr` we can use in our macro implementation.
        },
        other => {
            // In this case, the provided macro didn't expand into a literal - 
            // here you can report what happened to the user.
        },
    }
```

## Built-In Attributes
[guide-builtin]: #guide-builtin

Some attributes, like `#[path]` and `#[doc]`, are defined as part of the Rust language and are implemented by the compiler. However, they work very similarly to the process outlined above, and so you can use macro invocations in these with no issues.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The changes suggested by this RFC, as seen above, are:
- Extending the 'conventional' attribute grammar [described](https://docs.rs/syn/0.12/syn/struct.Attribute.html#syntax) in syn and in the compiler's AST, namely allowing [macro](https://docs.rs/syn/0.12/syn/struct.Macro.html) nodes wherever a literal is expected.
- Exposing a minimal API to user-defined procedural macros that allows expansion of macro invocations.
- Changing the built-in language attributes (as listed in the [reference](https://docs.rs/syn/0.12/syn/struct.Macro.html)) to similarly expand macro invocations in literal position.

Since macro definitions are added into the global scope as they are encountered in lexical order, any legal invocation will already be in scope for attributes like `#[path = ...]` and `#[cfg(...)]`. This means we don't need to worry about any [Russel's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox) issues where a macro prevents itself from being defined or in scope.

There is existing infrastructure in the compiler to resolve macros from their names to their definitions, and to expand invocations into their results. Parsing the token stream passed to `expand`, and tokenising the result, satisfies the required API.

# Drawbacks
[drawbacks]: #drawbacks

- Making it easier to programmatially control things like module locations may lead to 'code smells' like the example in the [summary](#summary).

- Allowing `#[path = env!(...)]` and `#[cfg(feature = env!(...))]` means cargo now needs to track environment variables as part of its dependency information.

- As discussed in the [comments](https://github.com/rust-lang/rfcs/pull/1990#issuecomment-305323449) of the RFC for `#[doc(include = ...)]` syntax, using macros in attributes is less appealing and ergonomic than adding a new `key = "value"` option. Do we want to encourage this?

# Rationale and alternatives
[alternatives]: #alternatives

- This is a problem that users will naturally encounter as their familiarity with macros and attributes increases. That this suddenly doesn't work is unexpected.
- After [rust-lang/rust#42164](https://github.com/rust-lang/rust/issues/42164) was fixed (some time before [rust-lang/rust/pull/47014](https://github.com/rust-lang/rust/pull/47014)), the 'trick' of indirectly using macro invocations through a layer of macro indirection now works:

```rust
macro_rules! linked {
  ($name:ident, $link_name:expr) => {
    #[link_section = $link_name]
    pub static $name: usize = 0;
  }
}

linked!(X, ".section_for_X");
linked!(Y, concat!(".section_for_", stringify!(Y)));
```

- We could choose that this be the accepted way of handling this combination of features. This is an uncomfortable friction whenever someone runs into it, and it feels like boilerplate for no benefit.

- We could delay providing a way for procedural macros to expand macro invocations and metavariables, instead just supporting them in a subset of built-in attributes.

# Unresolved questions
[unresolved]: #unresolved-questions

- What should a procedural macro API for macro invocation/metavariable expansion look like?
  - Should the macro scope and resolution information be hidden global state, as hypothesised [above](#guide-complication)? Should it be passed around as an opaque `Scope` struct, in preparation for a more expressive future API?
  - How would a third party implement something like `proc_macro2` for `expand`? Is there enough infrastructure in stable Rust to emulate it?
  - The `proc_macro2` crate is usable from outside a proc macro crate. Combined with `syn` and `quote` this makes exploring syntax manipulations very approachable, since we can simply set up a normal cargo binary project instead of a proc macro definition crate plus a proc macro caller crate.
  - How does this interact with the recent work on proc macro hygeine?

- Do we need to document how macros are implemented in more detail? How was [rust-lang/rust#42164](https://github.com/rust-lang/rust/issues/42164) resolved - in particular, what did it change about how metavariables are handled? Why doesn't it work for procedural macros ([issue pending](https://github.com/rust-lang/rust/issues/42164#issuecomment-355802889))?
