---
layout: post
excerpt_separator: <!--more-->
title: "Using Rust proc macros to explore the \"fully populated collection\" pattern"
permalink: proc-macros-fully-populated-collection/
date: 2023-07-11
---

This is an exploration of a pattern that I've only vaguely come across in whatever languages
(but may well be a known named thing?) that I'll call the "fully populated collection"
pattern, in the context of a Rust code-base

Also a bit of a tutorial on Rust [procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html),
being applied to this pattern-exploration

So we'll start with the conceptual exploration but if you prefer you can jump straight to the
[proc macro "intro" tutorial](#proc-macros) or [description of the proc macros for the
"fully populated collections" pattern](#fully-populated-collection-proc-macros)

## The "fully populated collection" pattern

In [`tree-sitter-grep`](https://github.com/helixbass/tree-sitter-grep) we currently have a
hard-coded list of supported "target languages" (eg Rust, Typescript, ...) and there are a
couple of collection data structures that should be populated with a corresponding value
"for every supported target language"

For example, we "lazily" try and parse the provided tree-sitter [query](https://tree-sitter.github.io/tree-sitter/using-parsers#pattern-matching-with-queries)
against different supported target languages if you don't explicitly specify the target
language (on the command-line)

Currently that is represented using a [`HashMap`](https://github.com/helixbass/tree-sitter-grep/blob/54a6a567cc7d46241e297de0362e7f247fc3d703/src/lib.rs#L87):
```
struct CachedQueries(HashMap<SupportedLanguageName, OnceLock<Result<Arc<Query>, QueryError>>>);
```

But so basically it's an invariant that that collection should always be populated with _all_ of the supported
target languages (with one key-value per supported target language)

So we currently just [iterate through all of the supported languages](https://github.com/helixbass/tree-sitter-grep/blob/54a6a567cc7d46241e297de0362e7f247fc3d703/src/lib.rs#L145)
when we construct the type:
```
impl Default for CachedQueries {
    fn default() -> Self {
        Self(
            ALL_SUPPORTED_LANGUAGES
                .iter()
                .map(|supported_language| (supported_language.name, Default::default()))
                .collect(),
        )
    }
}
```

And then the "access pattern" of the collection is just a normal `HashMap::get()` followed by a `.unwrap()`
(since we know that "every key" should be present)

But so there's a smell there, specifically the `.unwrap()` seems to be telling us "there's a missing abstraction"

A `HashMap` is designed for dynamic-sized collections with unknown keys

But what we have is really a fixed-size collection with a known "fixed"/"enumerable" set of keys

I'm not really familiar with where/if this concept "exists" in Rust-world

The closest thing I've probably encountered to it is in Typescript when you use a [`Record`](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type)
whose "key" type is eg a "union of string literals", eg from those docs:
```
interface CatInfo {
  age: number;
  breed: string;
}
 
type CatName = "miffy" | "boris" | "mordred";
 
const cats: Record<CatName, CatInfo> = {
  miffy: { age: 10, breed: "Persian" },
  boris: { age: 5, breed: "Maine Coon" },
  mordred: { age: 16, breed: "British Shorthair" },
};
```
And Typescript will yell at you if you try to instantiate that type without providing _all_ of the expected keys

So that's nice because the type system is statically guaranteeing the invariant that _all_ keys will be present
in any instance of that type, which seems like it should enable a `.get()`-style access pattern that doesn't
require a `.unwrap()` (ie `.get()` should return an `&T`, not an `Option<&T>`)

Ok so that's the starting point is basically thinking about "how could we encode that in Rust"

### Exploring the pattern in Rust

So I think there are various ways to probably think about encoding this pattern as an abstraction
in Rust. Where I landed was basically:
<ul>
  <li style="position: relative"> have a simple "token"<span class="sidenote-number"></span> <code>Copy</code> <code>enum</code> type that represents all of
    the ("enumerated") "keys" of these collections
    <span class="sidenote">"Token" is probably a terrible name for this, in the context of our collections it's really going to be more of a "key"/"index"?</span>
  </li>
  <li> represent the collections themselves as fixed-size arrays (that are indexed by converting the "token" <code>enum</code> &rarr;
    its <code>usize</code> equivalent)
  </li>
  <li> plan on using macros to encapsulate the creation of the token type + collections so that
    :hand-waves and mumbles something about statically guaranteed invariants:
  </li>
</ul>


So let's look at some bits of code from a "hand-written" (as opposed to macro-generated,
which we'll get into next) version of this

So the token type is just a simple typical [field-less enum](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/language.rs#L10):
```
#[derive(Copy, Clone, Debug, Eq, PartialEq, ValueEnum)]
pub enum SupportedLanguage {
    Rust,
    Typescript,
    ...
}
```
(where `ValueEnum` is [`clap::ValueEnum`](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html#enumerated-values),
which we're using to be able to parse a command-line `--language` argument.<br />
It's interesting that per the [`clap` docs](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html#enumerated-values)
(specifically the generated `ValueEnum::value_variants()` method there) its `ValueEnum` API
also seems to be pointing somewhat at this concept (by needing a run-time "fully-populated collection" of all of those
enum variants)?)

And then (imagining these things getting macro-generated) let's define a [newtype wrapper
type](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/language.rs#L47)
for the fixed-size collections:
```
pub struct BySupportedLanguage<T>([T; 22]);
```
(where `22` is the # of enum variants from the token `SupportedLanguage` type above)

And now let's create a couple of these fixed-size collections

Some of them (like `CachedQueries` above) don't have "unique values" per key, so instances of
those can just be initialized via typical "uniform array initialization", eg now for
`CachedQueries` we can [derive `Default`](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/lib.rs#L86):
```
#[derive(Default)]
struct CachedQueries(BySupportedLanguage<OnceLock<Result<Arc<Query>, QueryError>>>);
```

But then some of them do have unique values for each key, eg a mapping from supported target
language to corresponding [`tree_sitter::Language`](https://docs.rs/tree-sitter/latest/tree_sitter/struct.Language.html)

For the moment let's define it by just manually doing [array initialization of each element](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/language.rs#L200):
```
static SUPPORTED_LANGUAGE_LANGUAGES: Lazy<BySupportedLanguage<Language>> = Lazy::new(|| {
    BySupportedLanguage([
        tree_sitter_rust::language(),
        tree_sitter_typescript::language_tsx(),
        ...
    ])
});
```
(we're using [`once_cell::sync::Lazy`](https://docs.rs/once_cell/latest/once_cell/sync/struct.Lazy.html)
to initialize this `static` because I guess those `tree_sitter::Language`-creating methods aren't
`const` (if I understand `static` correctly)?)

But this is pretty sketchy because you have to make sure to list the values in the right order
(the type system will ensure that you supply the right _number_ of them, but has no idea how/if the
order corresponds to your "expected key ordering")

So ya we're definitely going to want to generate
that with a macro that is somehow aware of the correct ordering based on the "enumerated" keys

Ok there are some other "odds and ends" for making this fixed-size collection abstraction usable
(eg enabling easy token-type &#x2194; `usize` "massaging") but this gives a basic idea of where
we're landing

Before we move on though, let's look at the specific ergonomic reason that I added that newtype
wrapper around the actual `[T; 22]` fixed-size array type

#### Make it "feel" more like a `HashMap`

While the collection is technically an array, the idea is really that it's a "key-value" collection
(where the keys are our token type)

We can replace the smelly `.get(...).unwrap()` (of the `HashMap` version) with a simple ("infallible")
indexing access:
```
impl<T> Index<SupportedLanguage> for BySupportedLanguage<T> {
    type Output = T;

    fn index(&self, index: SupportedLanguage) -> &Self::Output {
        &self.0[index as usize]
    }
}
```

But then additionally there are places where we want to iterate over the keys and values of
these collections. Rather than doing eg `.enumerate()` + some clunky `usize` &rarr; token type conversion,
let's "treat it like a key-value collection" and expose `HashMap`-like iteration methods (on the
newtype wrapper type)

I won't show the whole code listing but [here](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/language.rs#L49)
is an implementation of `.iter()`/`.values()` for the newtype wrapper type

And [here](https://github.com/helixbass/tree-sitter-grep/blob/f672cca2e2646b690818e2ce7dac1dcdb6fe3f1e/src/lib.rs#L111)
is a place where the `CachedQueries` type uses that `.iter()` method to iterate over both the
token-type "pseudo-keys" and corresponding (`OnceLock<...>`) values of its fixed-size collection

Ok that's pretty cool, this isn't seeming like a total disaster. But we noted a couple places where
it's begging to be "made more legitimate" via macros

So let's (after perhaps a brief pause) roll up our sleeves and get into proc-macro world...




## <a name="proc-macros" /> Procedural macros: hmmmm hum hum are they hard

I've tended to "pull up short" of just going ahead and writing procedural macros
when they seem like they might be useful because I've felt a bit scared of how
to wrangle them

But I am starting to get to the point where I feel like I'll be able to "land it
relatively cleanly"

And in this case coming up with a rough sketch/plan for some procedural macros
to help with the fully-populated collections then came to fruition roughly how
I'd hoped

So if I can do it you probably can't because I'm much smarter than you!

No wait that's not it

Don't be skurred

But yes let's try and "drive from a place of clarity"

So in order to come up with a possibly high-confidence plan we need enough
of a mental model to reason about what/how we could express it using
procedural macros (or maybe we don't need procedural macros? But ya we kind
of do)



#### The mental model

A Rust macro basically gets the chance to replace the chunk of source code
where it's invoked with another chunk of source code

At the level that macros get their hands on that source code, though, it's
not guaranteed to actually be valid Rust code

What you get and give back are "token trees"

What are "token trees"?

Let's not actually even try and construct a cohesive mental model for that

In practical terms, we just need (a) to be able to wrangle (aka "parse") the
input token tree into something that holds all of the information we need
about how they invoked the macro

And then (b) we always want to output valid Rust code, we just have to know
how to construct that valid Rust code and return it as a token tree



### A really dumb function-style procedural macro

So let's narrow our scope

Procedural macros can be invoked via `#[derive(MyMacro)]`,
`#[my_macro(...)]`, or `my_macro!(...)`<span class="sidenote-number"></span>
<span class="sidenote">`my_macro![...]` and `my_macro! {...}` are just
other ways of writing this, from the macro's perspective it doesn't care
which one you used</span>

I'm pretty sure that for our purposes we will only need `my_macro!(...)`
"function-style" macros

So let's only focus on those

So the "input" (token tree) that our macro will get passed is everything between
the `(` and `)` in `my_macro!(...)`

To "flex against our mental model", let's try and write a procedural macro
that gets very angry at you unless you just pass it `"cheese"`, ie
`my_macro!("cheese")` is ok but `my_macro!(anything_else)` should fail in
whatever way procedural macros fail when you pass them things they didn't
expect

Ok cool where do we put it

Well hold on let's achieve clarity against our current mental model before
starting to write any actual code

That may seem hard given that we don't really know what procedural macro code
"looks like" but I'd argue no we can still reason about it

We know that we get handed a "token tree" of whatever is between the `(` and `)`

And we know that our job is to give back a token tree that's... valid Rust code
of some kind

Ok so here our job is to ascertain whether that "token tree" that we got passed
is... whatever the token tree for `"cheese"` looks like

Anything else would be uncivilized

And we can look forward to figuring out how to complain if that happens

And I didn't specify that I cared what happens after that, so let's assume we
can do some dumb sh** as far as what token tree we return

Ok but feel my flex you said we need to parse the
input token tree into something that holds all of the information we need
about how they invoked the macro

Juxtaposing this against the fundamental mental model
of [parse-don't-validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
that you want to come to mind whenever you see stuff like this,
I'd argue that in this case our "parsed representation" can be
completely empty aka `()`

Why because let's assume that we complain and give up _during parsing_
if what we see isn't `"cheese"`. But then at that point we don't need
any "representation" to know what state we're in. 100% of code paths
if we get that far are "we saw `"cheese"`". So it would be redundant
to encode that fact as "state" (eg a `struct` or even just a `bool`)

And that's in keeping with the spec that then we can output whatever we
want and noone will care

Ok I like it we're so into "parse-don't-validate" that we're still going
to think of it that way and it will feel very rigorous to describe it
as parsing into an empty (`()`) representation

Ok that flex is ready to pop we're going to parse-don't-validate the input
token tree `"cheese"` into the representation `()` or else we will complain
and give up

Now we're ready to write code



#### Where do you put a procedural macro?

I don't understand the "why" of this but I know the rule is "they have to go in
their own crate"

And I think that crate has to get "tagged" as being a special "procedural-macro" crate
but I forget where that goes, we'll pull up some existing example for reference to
sniff it out

But let's start with writing the core code and come back to the wiring-up

So create a new crate that will become our procedural macro crate<span class="sidenote-number"></span>:
<span class="sidenote">I legit have no idea whether there's a way to tell `cargo` "hey this is going
to be a procedural macro crate". Maybe that would make sense?</span>
```
$ cargo new --lib say_cheese
$ cd say_cheese
```
Ya I decided we will call our macro like `say_cheese!("cheese")` not `my_macro!("cheese")`

And now I will feed you the specific "signature" for our procedural macro (that we'll add to `src/lib.rs`):
```
use proc_macro::TokenStream;

#[proc_macro]
pub fn say_cheese(input: TokenStream) -> TokenStream {
    ...
}
```
I don't know where `proc_macro` (in `use proc_macro::TokenStream`) "comes from",
usually we either have to add something to `Cargo.toml` as a dependency or
else it has to come from `std::*` (or `alloc::*`/`core::*` I believe?)

But nope here "it's just there" somehow. Whatever makes sense maybe

Ok so `TokenStream` is our "token tree" input (and output) type that
we want to parse-don't-validate into `()`

Let's encapsulate that:
```
fn parse_input(input: TokenStream) -> () {
    unimplemented!()
}

#[proc_macro]
pub fn say_cheese(input: TokenStream) -> TokenStream {
    let parsed_nothing = parse_input(input);

    unimplemented!()
}
```

Ok my editor is yelling at me about `proc_macro` so
let's go ahead and inform it that this is a proc macro
crate

Looks to me like all we have to do to "make this a
special procedural-macro crate" is just make
sure this is in our `Cargo.toml`:
```
[lib]
proc-macro = true
```

And yup `cargo check` is passing with just a couple
unused-variable warnings

Great now clearly it is time to "poke at" (aka "parse")
that input `TokenStream`




#### Parsing the input

We could do an exploration of how to do that "manually from
scratch"

I don't actually know what that would look like

The reality of proc macros from what I've seen is that "everyone
uses [`syn`](https://docs.rs/syn) to parse the input"

So let's be practical and just sort of bake `syn` into our
mental model (or feel free to dig into the raw `TokenStream`
API if that's your style)

`$ cargo add syn`

So my mental model for `syn` is:

> It knows how to parse "chunks" of your token-tree input
> that actually look like chunks of Rust code into certain
> types that correspond to that type of thing in Rust code
> eg an `Expr` (for a Rust expression)

Ok cool so how easily does that apply in our case?

Well `"cheese"` is a chunk of Rust code, and in fact is an
expression

So let's just aim for that, we'll start by trying to leverage
`syn` to parse our input token tree into a `syn::Expr`

Ya let's temporarily change our spec to instead of `"cheese"`
we'll just complain if we get passed anything other than a Rust
expression

Then we can try testing it and seeing if it works and come
back to refining that to be `"cheese"`-specific

I will hold your hand of how to translate what we see in the
`syn` [docs](https://docs.rs/syn/2.0.25/syn/index.html)
as an example of parsing a "derive macro" input &rarr; our
"function-style macro" world:

We do the same exact thing we just say `as Expr` instead of
`as DeriveInput`:
```
use syn::{parse_macro_input, Expr};

fn parse_input(input: TokenStream) -> () {
    let parsed_expr = parse_macro_input!(input as Expr);
}
```
And actually `syn` is going to take care of the "complaining"
part for us (and do it in some way that should by default provide
helpful compiler error messages to the person trying to use this
procedural macro)

So I think we're kind of done

We can literally leave the code like that because `-> ()` is the same
as not returning anything

It will just presumably warn us about `parsed_expr` being unused
but we don't care

Ok on to testing... wait what's that my editor is yelling at me?

```
$ cargo check
error[E0308]: mismatched types
 --> src/lib.rs:5:23
  |
5 |     let parsed_expr = parse_macro_input!(input as Expr);
  |                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `()`, found `TokenStream`
  |
  = note: this error originates in the macro `parse_macro_input` (in Nightly builds, run with -Z macro-backtrace for more info)
```

Hmm ok I legitimately thought this was going to work

Ok after slight sniffing what's going on here is that
`parse_macro_input!(...)` is a _little_ more magic than I expected

Specifically it's hard-coding in the fact that it can `return`
a `TokenStream` if it fails to parse the input

So... we could either change our function signature to return `TokenStream`
to make it happy

Or we could not use `parse_macro_input!()`

I'm going with the latter because we want to return the incredibly meaningful
`()` not a `TokenStream`




#### Avoiding `parse_macro_input!()`

Ok so squinting at the definition of `parse_macro_input!()` it looks like
it's more or less just a thin wrapper around calling
[`syn::parse()`](https://docs.rs/syn/latest/syn/fn.parse.html)

And per why it was trying to do a `return` for us, that `syn::parse()` call
is fallible and returns a `syn::Result`

So the appropriate return type for our function is going to be `syn::Result<()>`

And we can write it like so:
```
use syn::parse;

fn parse_input(input: TokenStream) -> syn::Result<()> {
    let parsed_expr = parse::<Expr>(input)?;

    Ok(())
}
```

And then in our main `say_cheese()` function we can mimic that
"return if it couldn't parse it" behavior (from `parse_macro_input!()`):
```
pub fn say_cheese(input: TokenStream) -> TokenStream {
    let parsed_nothing = match parse_input(input) {
        Ok(data) => data,
        Err(err) => return err.to_compile_error().into(),
    };

    unimplemented!()
}
```

Ok nice good to poke under the hood of `parse_macro_input!()`

So I think we're ready to test it




#### Testing the macro

Let's create a new dummy crate for testing<span class="sidenote-number"></span>
<span class="sidenote">Yes writing "actual tests" sounds nice. I don't know what to reach for for testing
macros so let's just manually test</span>
our procedural macro:

```
$ cargo new test-say-cheese
```

We just add our proc macros crate as a dependency:
```
# test-say-cheese/Cargo.toml

[dependencies]
say_cheese = { path = "../say_cheese" }
```

And then try and pass some valid Rust expression to it and see if it complains:
```
// test-say-cheese/src/main.rs

use say_cheese::say_cheese;

fn main() {
    say_cheese!("Hello, world!");
}
```
```
$ cargo check
error: proc macro panicked
 --> src/main.rs:4:5
  |
4 |     say_cheese!("Hello, world!");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: message: not implemented
```

Ok fair we said we didn't care about what our proc macro
returns but the compiler cares a little bit

So what's the dumbest thing we can do to have our `say_cheese()`
function return a valid `TokenStream`?

Well currently we don't know how to create new `TokenStream`'s

So again feel free to dig into that yourself but I'm going to say
"see if we can just `.clone()` the `input` `TokenStream`":
```
// say_cheese/src/lib.rs

#[proc_macro]
pub fn say_cheese(input: TokenStream) -> TokenStream {
    let parsed_nothing = match parse_input(input.clone()) { // <-- added this .clone()
        Ok(data) => data,
        Err(err) => return err.to_compile_error().into(),
    };

    input // <-- and return this
}
```

Ok (from `test-say-cheese/`) <br />
`$ cargo run`

Fabulous

Does it complain as well as we expected if we change it to something
that can't be parsed as a Rust expression

```
// test-say-cheese/src/main.rs

fn main() {
    say_cheese!(->);
}
```
```
$ cargo run
error: unsupported expression; enable syn's features=["full"]
 --> src/main.rs:4:18
  |
4 |     say_cheese!(->);
  |                  ^
```

Ok (first) mission accomplished

So now we need to refine it to only accept `"cheese"`

To do that we're going to have to sniff at `syn`'s "parse-able" types
a little more closely




#### What does `syn` think a `"cheese"` is?

Ok looking at `syn`'s docs, I spot `ExprLit` which says it could be
(the parsed version of) `"foo"`

So that sounds like the "narrower" type than `syn::Expr` that we want
to try and parse into

That should get us as far as it complaining if we pass anything besides
a literal of some kind

But then to check if that literal is the literal `"cheese"` I think we
will have to do ourselves based on what we see in that `ExprLit`

So "drilling down" through its [docs](https://docs.rs/syn/latest/syn/struct.ExprLit.html),
it looks like its `lit` field needs to be a `Lit::Str` whose [`.value()`](https://docs.rs/syn/latest/syn/struct.LitStr.html#method.value)
is `"cheese"`:

```
// say_cheese/src/lib.rs

use syn::{ExprLit, Lit};

fn parse_input(input: TokenStream) -> syn::Result<()> {
    match parse::<ExprLit>(input)?.lit {
        Lit::Str(lit_str) if lit_str.value() == "cheese" => Ok(()),
        _ => // what do we do here?
    }
}
```

Ok right I guess we need to know how to complain ourselves

Reading `syn`'s [`Error` docs](https://docs.rs/syn/latest/syn/parse/struct.Error.html)
it looks like this is appropriate:

```
// say_cheese/src/lib.rs

use syn::Error;

fn parse_input(input: TokenStream) -> syn::Result<()> {
    match parse::<ExprLit>(input)?.lit {
        Lit::Str(lit_str) if lit_str.value() == "cheese" => Ok(()),
        lit => Err(Error::new(lit.span(), "expected \"cheese\"")), // <-- added this
    }
}
```

Ok so to test that first the "happy path":

```
// test-say-cheese/src/main.rs

fn main() {
    say_cheese!("cheese");
}
```
```
$ cargo run
```

...and the "unhappy path":
```
// test-say-cheese/src/main.rs

fn main() {
    say_cheese!("not cheese");
}
```
```
$ cargo run
error: expected "cheese"
 --> src/main.rs:4:17
  |
4 |     say_cheese!("not cheese");
  |                 ^^^^^^^^^^^^
```

I just wept at its beauty

Ok I'm going to call that "part 2"

And now for part 3 you can let me take the
wheel and spell out the plan + execution of
the "fully-populated collection" proc macros
while you bask in the glory of what just
happened




## <a name="fully-populated-collection-proc-macros" /> Some proc macros for "fully populated collections"

The "sketch" for how I'd like to be able to use these is<span class="sidenote-number"></span><span class="sidenote">I
called this `fixed_map!()`, but I'm liking the "fully populated collections" name, but this macro really just
generates the "token" (which per above is also a bad name) type and _another_ macro for actually creating the
collections...</span>:
```
fixed_map! {
    name => SupportedLanguage,
    variants => [
        Rust,
        Typescript,
        ...
    ]
}
```
and then for fully-populated collections of that key type with "unique values":
```
let whatever = by_supported_language!(
    Rust => tree_sitter_rust::language(),
    Typescript => tree_sitter_typescript::language_tsx(),
    ...
);
```
(where `by_supported_language!()` is a macro that gets generated by `fixed_map!()`)

To implement `fixed_map!()` we'll need to "cross the bridge" of parsing "custom non-Rust syntax"
(eg `name => ...`)

But really the tricky thing is that `by_supported_language!()` looks like it needs to
be a procedural macro, I don't think you could make that work as a purely declarative macro

But... I don't think you could just have one procedural macro generate another procedural macro
that you can then "immediately use" in the same file like that (because procedural macros need
to be defined in their own crate)

So the trick is that there will be a "generic" version of `by_supported_language!()` that's a
procedural macro whose job is to create fully populated collection instances, call it `by_fixed_map!()`

And it will have to be "parameterized" with the stuff that's specific to _that_ fully-populated collection

And we can make the generated _specific_ version eg `by_supported_language!()` be a _declarative_
macro that like "hard-codes"/"pre-curries" those "specific" parameters in its definition and passes them
to `by_fixed_map!()`




#### Implementation

A type to represent the parsed version of the arguments to `fixed_map!()`:
```
use syn::Ident;

struct FixedMapArgs {
    name: Ident,
    variants: Vec<Ident>,
}
```

And for the actual parsing of it I just sort of "jimmied" this together:
```
use syn::{
    parse::{Parse, ParseStream},
    Token, Expr, ExprArray,
};

fn expr_to_ident(expr: Expr) -> Ident {
    match expr {
        Expr::Path(expr) => expr.path.get_ident().unwrap().clone(),
        _ => panic!("Expected Ident"),
    }
}

fn parse_idents_array(input: ParseStream) -> syn::Result<Vec<Ident>> {
    Ok(input
        .parse::<ExprArray>()?
        .elems
        .into_iter()
        .map(expr_to_ident)
        .collect())
}

impl Parse for FixedMapArgs {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        let mut name: Option<Ident> = Default::default();
        let mut variants: Option<Vec<Ident>> = Default::default();
        while !input.is_empty() {
            let key: Ident = input.parse()?;
            input.parse::<Token![=>]>()?;
            match &*key.to_string() {
                "name" => {
                    name = Some(input.parse::<Ident>()?);
                }
                "variants" => {
                    variants = Some(parse_idents_array(input)?);
                }
                _ => panic!("didn't expect key {}", key),
            }
            input.parse::<Token![,]>()?;
        }
        Ok(FixedMapArgs {
            name: name.expect("Expected name specifier"),
            variants: variants.expect("Expected variants specifier"),
        })
    }
}
```

Then for generating the output, we didn't get into this above but
similarly to how "everyone uses `syn` for parsing the input", it seems
that "everyone uses [`quote`](https://docs.rs/quote) for generating the output"

And `quote!()` is sort of "template"-y

I won't show all of the generated stuff but here is an example of
generating the actual "token" `enum` type definition:
```
#[proc_macro]
pub fn fixed_map(input: TokenStream) -> TokenStream {
    let FixedMapArgs { name, variants } = parse_macro_input!(input as FixedMapArgs);


    let token_enum_definition = get_token_enum_definition(&name, &variants);
    ...

    quote! {
        #token_enum_definition
        ...
    }
    .into()
}

fn get_token_enum_definition(name: &Ident, variants: &[Ident]) -> proc_macro2::TokenStream {
    quote! {
        #[derive(Copy, Clone, Debug, Eq, PartialEq, clap::ValueEnum)]
        pub enum #name {
            #(#variants),*
        }
    }
}
```
so that will generate in our case the
```
#[derive(...)]
pub enum SupportedLanguage {
    Rust,
    Typescript,
    ...
}
```

Ok let's take a look at the tricky part

First the "generic" `by_fixed_map!()` proc macro

We'll want to specifically invoke it like:
```
by_fixed_map!(
    // this is the supplied argument to `by_supported_language!()`
    [
        Rust => tree_sitter_rust::language(),
        Typescript => tree_sitter_typescript::language_tsx(),
        ...
    ],
    // this is a list of all of the variants in the "right order"
    // this will be "hard-coded" in `by_supported_language!()`
    [
        Rust,
        Typescript,
        ...
    ],
    // this is the name of the "fully-populated collection type"
    // this will also be "hard-coded" in `by_supported_language!()`
    BySupportedLanguage
)
```

Ok its whole implementation:

```
struct ByFixedMapArgs {
    value_mapping: HashMap<Ident, Expr>,
    variants: Vec<Ident>,
    collection_type_name: Ident,
}

impl Parse for ByFixedMapArgs {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        let value_mapping_content;
        bracketed!(value_mapping_content in input);
        let mut value_mapping: HashMap<Ident, Expr> = Default::default();
        while !value_mapping_content.is_empty() {
            let key: Ident = value_mapping_content.parse()?;
            value_mapping_content.parse::<Token![=>]>()?;
            let value: Expr = value_mapping_content.parse()?;
            if value_mapping.contains_key(&key) {
                panic!("Repeated key: {key}");
            }
            value_mapping.insert(key, value);
            if value_mapping_content.is_empty() {
                break;
            }
            value_mapping_content.parse::<Token![,]>()?;
        }
        input.parse::<Token![,]>()?;
        let variants = parse_idents_array(input)?;
        input.parse::<Token![,]>()?;
        let collection_type_name: Ident = input.parse()?;
        Ok(Self {
            value_mapping,
            variants,
            collection_type_name,
        })
    }
}

#[proc_macro]
pub fn by_fixed_map(input: TokenStream) -> TokenStream {
    let ByFixedMapArgs {
        value_mapping,
        variants,
        collection_type_name,
    } = parse_macro_input!(input as ByFixedMapArgs);

    if value_mapping.len() != variants.len() {
        panic!("Incorrect variants");
    }

    let ordered_values = variants
        .iter()
        .map(|variant| value_mapping.get(variant).expect("Incorrect variants"))
        .collect::<Vec<_>>();

    quote! {
        #collection_type_name([
            #(#ordered_values),*
        ])
    }
    .into()
}
```

So again some somewhat-fiddly manual parsing,
specifically [`syn::bracketed!()`](https://docs.rs/syn/latest/syn/macro.bracketed.html)
came in handy

And then we are just driving generation of an array literal whose elements
are in the "correct" order as specified by the "variants" argument

And then per above the piece that wires them together is to have `fixed_map!()` generate
a procedural macro that hard-codes the "variants" and "collection type name" arguments:

```
use quote::format_ident;

#[proc_macro]
pub fn fixed_map(input: TokenStream) -> TokenStream {
    let FixedMapArgs { name, variants } = parse_macro_input!(input as FixedMapArgs);

    let collection_type_name = format_ident!("By{name}"); // <-- eg "BySupportedLanguage"
    ...
    let collection_generator_macro_definition =
        get_collection_generator_macro_definition(&collection_type_name, &variants);

    quote! {
        ...
        #collection_generator_macro_definition
    }
    .into()
}

fn get_collection_generator_macro_definition(
    collection_type_name: &Ident,
    variants: &[Ident],
) -> proc_macro2::TokenStream {
    let macro_name = format_ident!("{}", collection_type_name.to_string().to_snake_case());
    quote! {
        #[macro_export]
        macro_rules! #macro_name {
            ($($variant:ident => $value:expr),* $(,)?) => {
                proc_macros::by_fixed_map!(
                    [$($variant => $value),*],
                    [
                        #(#variants),*
                    ],
                    #collection_type_name
                )
            }
        }
    }
}
```

And it works!

If you want to see the actual code, it was split across
[these](https://github.com/helixbass/tree-sitter-grep/pull/52)
[two](https://github.com/helixbass/tree-sitter-grep/pull/53)
PR's




### In conclusion

There wasn't some major ergonomic nor performance reason to have to
move to the "fully-populated collection" pattern<span class="sidenote-number"></span>
<span class="sidenote">Make no mistake, this was "a version" of taking a stab at
what implementing that pattern in Rust could look like, I think there could be other
(possibly already-existing) approaches for how it might look</span>
for the `SupportedLanguage` stuff in the `tree-sitter-grep` code-base

But I like that it feels like we're using more appropriate/descriptive data structures
(and that should translate to higher efficiency even if perhaps negligible
in the context of `tree-sitter-grep` - look Ma
no allocations!)

And it was a good excuse to dig into procedural macros a little bit
