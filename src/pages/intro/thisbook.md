## About this book

This book is divided into two parts.

In Part I we introduce *type class derivation*,
which allows us to create type class instances
for any algebraic data type
using only a handful of generic rules.
Part I consists of four chapters:

  - In Chapter [@sec:representations]
    we introduce *generic representations*
    and shapeless' `Generic` type class,
    which can produce a generic encoding
    for any case class or sealed trait.

  - In Chapter [@sec:generic] we use `Generic`
    to derive instances of a custom type class.
    We use CSV encoding as an example,
    but the techniques we cover
    can be extended to many situations.
    We also introduce shapeless' `Lazy` type,
    which lets us handle recursive data like lists and trees.

  - In Chapter [@sec:type-level-programming] we cover some more theory:
    dependent types, dependently typed functions,
    and type level programming.
    We introduce the programming patterns
    we need to generalise the techniques from earlier chapters
    to more advanced applications of shapeless.

  - In Chapter [@sec:labelled-generic] we introduce `LabelledGeneric`,
    a variant of `Generic` that exposes field and type names
    as part of its generic representations.
    We also introduce some new theory:
    literal types, singleton types, phantom types, and type tagging.
    We demonstrate `LabelledGeneric` by creating
    a JSON encoder that preserves field and type names in its output.

In Part II we introduce the "ops type classes"
provided in the `shapeless.ops` package.
The ops type classes form a vast library of tools
for manipulating generic representations.
Rather than discuss every op in detail,
we provide a theoretical primer in three chapters:

  - In Chapter [@sec:ops] we discuss
    the general layout of the ops type classes
    and provide an example
    that strings several simple ops together
    to form a powerful "case class migration" tool.

  - In Chapter [@sec:poly] we introduce
    *polymorphic functions* in shapeless,
    also known as `Polys`,
    and show how they are used in
    ops type classes for `mapping`,
    `flatMapping` and `folding`
    over generic representations.

  - Finally, in Chapter [@sec:nat] we introduce
    the `Nat` type that shapeless uses
    to represent natural numbers at the type level.
    We introduce several related ops type classes,
    and use `Nat` to develop
    our own version of Scalacheck's `Arbitrary`.

<div class="callout callout-info">
*Source code for examples*

We've placed source code for
many of the examples in this guide
in the accompanying Github repo:

`https://github.com/underscoreio/shapeless-guide-code`

The `exercises` branch contains
skeleton Scala files with `TODOs` to fill in,
and the `solutions` branch contains
completed implementations.
</div>
