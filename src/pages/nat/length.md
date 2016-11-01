## Length of generic representations

One use case for `Nat` is
determining the length of `HLists` and `Coproducts`.
Shapeless provides the
`shapeless.ops.hlist.Length` and
`shapeless.ops.coproduct.Length` type classes for this.

Because of the similarity of the names,
we typically import the `hlist` and `coproduct` packages
refer to the relevant type classes as `package.Length`:

```tut:book:silent
import shapeless._
import shapeless.ops.nat.ToInt
import shapeless.ops.{hlist, coproduct}
```

```tut:book
val hlistLength = hlist.Length[String :: Int :: Boolean :: HNil]
val coproductLength = coproduct.Length[Double :+: Char :+: CNil]
```

Instances of `Length` have a type member `Out`
that represents the length as a `Nat`.
We can either summon an instance of `ToInt` ourselves:

```tut:book
implicitly[ToInt[hlistLength.Out]].apply()
```

or use the `Nat.toInt` helper:

```tut:book
Nat.toInt[coproductLength.Out]
```

Let's use this in a concrete example.
We'll create a `SizeOf` type class that
counts the number of fields in a case class
and exposes it as a simple `Int`:

```tut:book:silent
trait SizeOf[A] {
  def value: Int
}

def sizeOf[A](implicit size: SizeOf[A]): Int =
  size.value
```

To create an instance of `SizeOf` we need three things:

1. a `Generic` to calculate the corresponding `HList` type;
2. a `Length` to calculate the length of the `HList` as a `Nat`;
3. a `ToInt` to convert the `Nat` to an `Int`.

Here's a working implementation
written in the style described in Chapter [@sec:type-level-programming]:

```tut:book:silent
implicit def genericSizeOf[A, L <: HList, N <: Nat](
  implicit
  generic: Generic.Aux[A, L],
  size: hlist.Length.Aux[L, N],
  sizeToInt: ToInt[N]
): SizeOf[A] =
  new SizeOf[A] {
    val value = sizeToInt.apply()
  }
```

We can test our code as follows:

```tut:book:silent
case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

```tut:book
sizeOf[IceCream]
```
