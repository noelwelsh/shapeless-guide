## Debugging implicit resolution

```tut:book:invisible
import shapeless._

trait CsvEncoder[A] {
  val width: Int
  def encode(value: A): List[String]
}

def createEncoder[A](w: Int)(func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    val width = w
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(1)(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(1)(num => List(num.toString))

implicit val doubleEncoder: CsvEncoder[Double] =
  createEncoder(1)(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(1)(bool => List(if(bool) "yes" else "no"))

implicit def optionEncoder[A](implicit encoder: CsvEncoder[A]): CsvEncoder[Option[A]] =
  createEncoder(encoder.width)(opt => opt.map(encoder.encode).getOrElse(List.fill(encoder.width)("")))

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(0)(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case h :: t =>
      hEncoder.value.encode(h) ++ tEncoder.encode(t)
  }

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(0)(cnil => ???)

implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case Inl(h) => hEncoder.value.encode(h) ++ List.fill(tEncoder.width)("")
    case Inr(t) => List.fill(hEncoder.value.width)("") ++ tEncoder.encode(t)
  }

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  enc: Lazy[CsvEncoder[R]]
): CsvEncoder[A] =
  createEncoder(enc.value.width)(a => enc.value.encode(gen.to(a)))
```

Failures in implicit resolution
can be confusing and frustrating.
Here are a couple of techniques to use
when implicits go bad.

### Debugging using implicitly

What can we do when the compiler
simply fails to find an implicit value?
The failure could be due to any one of
the implicit resolution rules in use.
For example:

```tut:book:silent
case class Foo(bar: Int, baz: Float)
```

```tut:book:fail
implicitly[CsvEncoder[Foo]]
```

The reason for the failure is that
we haven't defined an `Ordering` for `Float`.
However, this may not be obvious in application code.
We can work through the expected expansion sequence
to find the source of the error,
inserting calls to `implictly`
above the error to see if they compile.
We start with the generic representation of `Foo`:

```tut:book:fail
implicitly[CsvEncoder[Int :: Float :: HNil]]
```

This fails to compile so we know
we can ignore `genericEncoder` for now.
The next step is to try the components of the `HList`:

```tut:book:silent
implicitly[CsvEncoder[Int]]
```

```tut:book:fail
implicitly[CsvEncoder[Float]]
```

`Int` passes but `Float` fails.
`CsvEncoder[Float]` is a leaf in our tree of expansions,
so we know to start by
implementing this missing type class instance.
If adding the instance doesn't fix the problem
we repeat the process to find the next point of failure.

### Debugging using reify

The `reify` method from `scala.reflect`
takes a Scala expression as a parameter and returns
an AST note representing the expression tree,
complete with type annotations:

```tut:book:silent
import scala.reflect.runtime.universe._
```

```tut:book
println(reify(implicitly[CsvEncoder[Int]]))
```

The types inferred during implicit resolution
can give us hints about problems.
After implicit resolution,
any remaining existential types such as `A` or `T`
provide a sign that something has gone wrong.
Similarly, "top" and "bottom" types such as `Any` and `Nothing`
are evidence of failure.
