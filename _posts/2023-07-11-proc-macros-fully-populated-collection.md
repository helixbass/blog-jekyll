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
[proc macro stuff](#proc-macros)

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

Great now clearly it is time to "poke at" (aka "parse")
that input `TokenStream`

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

Then we can try wiring it up and seeing if it works and come
back to refining that to be `"cheese"`-specific




#### Using `syn` for parsing the input

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

Ok on to the "wiring-up"



#### Wiring up a procedural macro

Looks to me like all we have to do to "make this a special procedural-macro crate" is just make
sure this is in our `Cargo.toml`:
```
[lib]
proc-macro = true
```

Let's assume that's good to go and create a new dummy crate for testing<span class="sidenote-number"></span>
<span class="sidenote">Yes writing "actual tests" sounds nice. I don't know what to reach for for testing
macros so let's just manually test</span>
our procedural macro:

```
$ cargo new test-say-cheese
```

#### Upsides
#### Downsides
