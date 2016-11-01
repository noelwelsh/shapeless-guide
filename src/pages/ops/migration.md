## Case study: case class migrations {#sec:ops:migration}

The power of ops type classes fully crystalizes
when we chain them together
as building blocks for our own code.
We'll finish this chapter with a powerful example:
a type class for performing "migrations"
(aka "evolutions") on case classes[^database-migrations].
For example, if version 1 of our app has a case class:

[^database-migrations]: The term is stolen from
"database migrations"---SQL scripts that
automate upgrades to a database schema.

```tut:book:silent
case class IceCreamV1(name: String, numCherries: Int, inCone: Boolean)
```

our type class should let us perform certain
mechanical "upgrades" for free:

```tut:book:silent
// Remove fields:
case class IceCreamV2a(name: String, inCone: Boolean)

// Reorder fields:
case class IceCreamV2b(name: String, inCone: Boolean, numCherries: Int)

// Insert fields (provided we can determine a default value):
case class IceCreamV2c(
  name: String, inCone: Boolean, numCherries: Int, numWaffles: Int)
```

Ideally we'd like to be able to write code like this:

```scala
IceCreamV1("Sundae", 1, false).migrateTo[IceCreamV2a]
```

The type class should take care of the migration
without additional boilerplate.

### The type class

The `Migration` type class represents
a transformation from a source to a destination type.
Both of these are going to be "input" types in our derivation,
so we model both as type parameters.
We don't need an `Aux` type alias
because there are no type members to expose:

```tut:book:silent
trait Migration[A, B] {
  def apply(a: A): B
}
```

We'll introduce an extension method now
to make later examples easier to read:

```tut:book:silent
implicit class MigrationOps[A](a: A) {
  def migrateTo[B](implicit migration: Migration[A, B]): B =
    migration.apply(a)
}
```

### Step 1. Removing fields

Let's build up the solution piece by piece,
starting with field removal.
We can do this in several steps:

 1. convert `A` to its generic representation;
 2. filter the `HList` from step 1---only retain
    fields that are also in `B`;
 3. convert the output of step 2 to `B`.

We can implement steps 1 and 3 with `Generic` or `LabelledGeneric`,
and step 2 with an op called `Intersection`:

```tut:book:silent
import shapeless._
import shapeless.ops.hlist

implicit def genericMigration[A, B, ARepr <: HList, BRepr <: HList](
  implicit
  aGen    : Generic.Aux[A, ARepr],
  bGen    : Generic.Aux[B, BRepr],
  inter   : hlist.Intersection.Aux[ARepr, BRepr, BRepr]
): Migration[A, B] = new Migration[A, B] {
  def apply(a: A): B =
    bGen.from(inter.apply(aGen.to(a)))
}
```

Take a moment to locate [`Intersection`][code-ops-hlist-intersection]
in the shapeless codebase.
Its `Aux` type alias takes three parameters:
two input `HLists` and one output for the intersection type.
In the example above we are specifying
`ARepr` and `BRepr` as the input types
and `BRepr` as the output type.
This means implicit resolution will only succeed
if `B` has an exact subset of the fields of `A`,
specified with the exact same names in the same order:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
```

We get a compile error if
we try to use `Migration` with non-conforming types:

```tut:book:fail
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]
```

<div class="callout callout-info">
*Field uniqueness*

`Intersection` doesn't use the field names from `LabelledGeneric`.
What would have happened if we had used `Generic` above instead?

Although we're not using the field names directly,
they are having an effect on the uniqueness checks
`Intersection` uses to identify fields.
If we had used regular `Generic`,
the behaviour would be different and likely incorrect.
</div>

### Step 2. Reordering fields

We need to lean on another ops type class
to add support for reordering.
[`hlist.Align`][code-ops-hlist-align]
allows us reorder the fields in one `HList`
to match the order the appear in another `HList`.
We can redefine our instance using `Align` as follows:

```tut:book:silent
implicit def genericMigration[
  A, B,
  ARepr <: HList, BRepr <: HList,
  Unaligned <: HList
](
  implicit
  aGen    : LabelledGeneric.Aux[A, ARepr],
  bGen    : LabelledGeneric.Aux[B, BRepr],
  inter   : hlist.Intersection.Aux[ARepr, BRepr, Unaligned],
  align   : hlist.Align[Unaligned, BRepr]
): Migration[A, B] = new Migration[A, B] {
  def apply(a: A): B =
    bGen.from(align.apply(inter.apply(aGen.to(a))))
}
```

We introduce a new type parameter called `Unaligned`
to represent the intersection of `ARepr` and `BRepr`
before alignment,
and use `Align` to convert `Unaligned` to `BRepr`.
With this modified type class instance
we can support the removal and reordering of fields:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]

```

However, we still get a failure if we try to add fields:

```tut:book:fail
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2c]
```

### Step 3. Adding new fields

We need a mechanism for calculating default values
to suppose the addition of new fields.
Shapeless doesn't provide a type class for this
so we'll make our own:

```tut:book:silent
trait Empty[A] {
  def get: A
}

def createEmpty[A](body: => A): Empty[A] =
  new Empty[A] {
    def get: A = body
  }
```

We can define instances for `Empty`
using the techniques from Chapter [@sec:labelled-generic].
We need instances for a few core types,
`HNil`, and `HLists` with tagged heads:

```tut:book:silent
import shapeless.labelled.{field, FieldType}

implicit val emptyInt     : Empty[Int]     = createEmpty(0)
implicit val emptyDouble  : Empty[Double]  = createEmpty(0.0)
implicit val emptyBoolean : Empty[Boolean] = createEmpty(false)
implicit val emptyString  : Empty[String]  = createEmpty("")
implicit val emptyHnil    : Empty[HNil]    = createEmpty(HNil)
implicit def emptyHList[K <: Symbol, H, T <: HList](
  implicit
  hEmpty: Lazy[Empty[H]],
  tEmpty: Empty[T]
): Empty[FieldType[K, H] :: T] =
  createEmpty(field[K](hEmpty.value.get) :: tEmpty.get)
```

We need to combine `Empty` with a couple of other ops
to complete our final implementation of `Migration`.
Here's the full list of steps:

 1. use `LabelledGeneric` to convert `A` to its generic representation;
 2. use `Intersection` to calculate an `HList` of fields common to `A` and `B`;
 3. calculate the types of fields that appear in `B` but not in `A`;
 4. use `Empty` to calculate a default value of the type from step 3;
 5. append the common fields from step 2 to the new field from step 4;
 6. use `Align` to reorder the fields from step 5 in the same order as `B`;
 7. use `LabelledGeneric` to convert the output of step 6 to `B`.

We've already seen how to implement steps 1, 2, 4, 6, and 7.
We can implement step 3 using an ops type class called `Diff`
that works very similarly to `Intersection`.
We can implement step 5 using another ops type class called `Prepend`.
Here's the complete solution:

```tut:book:silent
implicit def genericMigration[
  A, B, ARepr <: HList, BRepr <: HList,
  Common <: HList, Added <: HList, Unaligned <: HList
](
  implicit
  aGen    : LabelledGeneric.Aux[A, ARepr],
  bGen    : LabelledGeneric.Aux[B, BRepr],
  inter   : hlist.Intersection.Aux[ARepr, BRepr, Common],
  diff    : hlist.Diff.Aux[BRepr, Common, Added],
  empty   : Empty[Added],
  prepend : hlist.Prepend.Aux[Added, Common, Unaligned],
  align   : hlist.Align[Unaligned, BRepr]
): Migration[A, B] =
  new Migration[A, B] {
    def apply(a: A): B =
      bGen.from(align(prepend(empty.get, inter(aGen.to(a)))))
  }
```

Note that we don't end up using
every type class at the value level.
We use `Diff` to calculate the `Added` data type,
but we don't actually need `diff.apply` at run time.
Instead we use `Empty` to summon an instance of `Added`.

With this final version of the type class instance in place
we can use `Migration` for all the use cases we set out
at the beginning of the case study:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2c]
```

It's amazing what we can create with ops type classes.
`Migration` has a single `implicit def`
with a single line of value-level implementation.
It allows us to automate migrations between *any* pair of case classes,
in roughly the same amount of code we'd write
to handle a *single* pair of types using the standard library.
