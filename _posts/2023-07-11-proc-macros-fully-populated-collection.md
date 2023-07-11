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
  <li> represent the collections themselves as fixed-size arrays (that are indexed by converting the "token" <code>enum</code> ->
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

<a name="proc-macros" />

#### Upsides
#### Downsides
