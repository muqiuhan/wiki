#scala #fp #typesystem

When familiarizing myself with additions in Scala 3, it was improvements in meta-programming capabilities that caught my eye. I wondered what it would take to implement a simple refinement types library. It is definitely not my plan to end up with a full-blown refinement library as [refined](https://github.com/fthomas/refined), but only to understand if the language's new version can offer some improvements in the process.

We will use literal types as a basic construct. The introduction of literal types was a subject of [SIP-23](https://docs.scala-lang.org/sips/42.type.html), and they've been already included in Scala 2.13. They enable you to use literals of primitive types at places where types are expected:

```
val a: 4 = 4
// val b: 5 = a // fails compilation with: 
// Found:    (a : (4 : Int))
// Required: (5 : Int)
```

What Scala 3 adds to the mix are compile-time operators on literal types. You can found them in the `scala.compiletime` package:

```
type PlusTwo[T <: Int] = scala.compiletime.ops.int.+[2, T]

val a: PlusTwo[4] = 6
// val bb: PlusTwo[4] = 7 // fails compilation with:
// Found:    (7 : Int)
// Required: (6 : Int)
```

In the definition of `PlusTwo` I wanted to stress that `+` is a type operator, hence the prefix notation. In practice infix operator might be more convenient:

```
import scala.compiletime.ops.int.*

type PlusTwo[T <: Int] = 2 + T
```

One of operators available in `compiletime` is comparison operator `<` which looks particularly useful in scope of refinement types:

```
type <[X <: Int, Y <: Int] <: Boolean
```

Here are some examples of its usage:

```
val a: 5 < 10 = true
// val b: 15 < 10 = true // fails compilation with:
// Found:    (true : Boolean)
// Required: (false : Boolean)
```

However, in context of refinement types we would like to use `<` as a kind of type bound as opposed to just an operator returning Boolean. Therefore, while `<` is definitely helpful, it's not sufficient by itself to express refinement types. We're striving for something akin to:

```
// val a: Int < 10 = 5 // fails compilation
```

### Iteration 1

To be able to use comparison operators as a type bound we have to build some minimal structure:

```
trait Validated[PredRes <: Boolean]
given Validated[true] = new Validated[true] {}

trait RefinedInt[Predicate[_ <: Int] <: Boolean]
def validate[V <: Int, Predicate[_ <: Int] <: Boolean]
    (using Validated[Predicate[V]]): RefinedInt[Predicate] = new RefinedInt {}
```

The idea behind this is that code invoking `validate` will compile only if `Predicate[V]` evaluates to type `true`. Therefore, the whole business of validation will be offloaded to the second type parameter of `validate`.

Such minimal structure is enough to express something like this:

```
type LowerThan10[V <: Int] = V < 10
val lowerThan10: RefinedInt[LowerThan10] = validate[4, LowerThan10]
```

An equivalent written with type lambda:

```
val lowerThan10: RefinedInt[[V <: Int] =>> V < 10] = validate[4, [V <: Int] =>> V < 10]
```

I must admit the latter looks uglier, but in general, it is preferred as it allows us to avoid coming up with an unnecessary type name.

If you want to see how type lambdas are being used in the wild, I recommend taking a look at [type-level implementations](https://github.com/lampepfl/dotty/blob/master/library/src/scala/Tuple.scala#L139) of `scala.Tuple` higher kinded types of `Filter` or `Fold`.

This encoding, while very simplistic and not the most convenient to use, is quite flexible. Thanks to operators in `compiletime.ops.bool` we can build more complicated predicates as the following:

```
import scala.compiletime.ops.boolean.*

validate[7, [V <: Int] =>> V > 5 && V < 10]
```

If we try to pass incorrect input we will get a compilation error:

```
validate[4, [V <: Int] =>> V > 5 && V < 10]
// no implicit argument of type iteration1.Validated[(false : Boolean)] was found for parameter x$2 of method validate in package iteration1
// L25:   validate[4, [V <: Int] =>> V > 5 && V < 10]
```

The compilation error is not very helpful, especially compared to the message produced by [refined](https://github.com/fthomas/refined):

```
Left predicate of ((4 > 5) && (4 < 10)) failed: Predicate failed: (4 > 5).
```

It's something we will work on in iteration 2. That being said, with just a few lines of code we were able to get that basic version working.

### Iteration 2 - self-descriptive compilation errors

Mechanisms we've used so far, `compiletime` operators and implicit resolution, are not enough to implement friendly validation errors. That's because the result of implicit resolution is binary - either the implicit had been found or not. We need richer information in case of failure.

To do that, we will explore another new feature of Scala 3, which is `inline`.

First, we need to define an ADT for predicates:

```
sealed trait Pred
class And[A <: Pred, B <: Pred]         extends Pred
class Leaf                              extends Pred
class LowerThan[T <: Int & Singleton]   extends Leaf
class GreaterThan[T <: Int & Singleton] extends Leaf
```

The only notable thing in the above is mixing in `Singleton`. It restricts type `T` into being a singleton type, so that `LowerThan[Int]` will not compile.

Then, we have to interpret this ADT at compile-time:

```
import scala.compiletime.*
import scala.compiletime.ops.int.*

trait Validated[E <: Pred]

implicit inline def mkVal[V <: Int & Singleton, E <: Pred](v: V): Validated[E] =
  inline erasedValue[E] match
    case _: LowerThan[t] =>
      inline if constValue[V] < constValue[t]
        then new Validated[E] {}
        else
          inline val vs    = constValue[ToString[V]]
          inline val limit = constValue[ToString[t]]
          error("Validation failed: " + vs + " < " + limit)
    case _: GreaterThan[t] => // ommited here since it's symmetrical to LowerThan
    case _: And[a, b] =>
      inline mkVal[V, a](v) match
        case _: Validated[_] =>
          inline mkVal[V, b](v) match
            case _: Validated[_] => new Validated[E] {}
```

There are a few things worth noting here:

- `mkVal` has an `inline` modifier which tells the compiler that it should inline any invocation of this method at compile-time. If it's not possible, compiler will fail the compilation

- `erasedValue` comes from `compiletime` package. It's usually used in tandem with inline match. It allows us to match on the expression type, but we cannot access extracted value as that code is executed at compile-time

- You could have noticed a lower-case letter used for the type parameter in `case _: LowerThan[t]`, something against the usual convention. That was not a choice though. In Scala 3 you must use a lower-case identifier for a type being extracted from a pattern match. Using `case _: LowerThan[T]` would mean that the match would succeed only if `LowerThan` is parametrized with an already known type `T`. I like to compare that to term-level pattern match in which there's also a distinction between `case a =>` and ``case `a` =>``, which in regards to types becomes `case _: V[a]` and `case _: V[A]` respectively

- `constValue` comes from `compiletime` too. It returns the value of a singleton type

- `ToString` is a type-level counterpart of `toString` available only for singleton types of Int, so that `val a: ToString[5] = "5"` holds

- Calling `error` fails the compilation with provided message. In Scala 3.0.0 it cannot be invoked with interpolated string, yet it might be [possible in future](https://github.com/lampepfl/dotty/issues/10315).

- `mkVal` is defined as an implicit conversion so it will never be called explicitly

Once you got acquainted with these new Scala constructs, the code should not be hard to follow. The great news is that it's all it takes to have reasonable refinements types for Int [1](https://msitko.pl/blog/build-your-own-refinement-types-in-scala3.html#footnote1). Let's try it out:

```
val a: Validated[LowerThan[10]] = 6
val b: Validated[GreaterThan[5] And LowerThan[10]] = 6

// val y: Validated[GreaterThan[5] And LowerThan[10]] = 1
// fails with:
// Validation failed: 1 > 5
```

#### A note on testing

Since the core functionality of refined boils down to preventing some code from being compiled, we have to specify negative test cases as code snippets that do not compile. In Scala 3 there's a built-in operation for that: `scala.compiletime.testing.typeCheckErrors`. We can employ it to write assertions:

```
import scala.compiletime.testing.typeCheckErrors

class IntSpec extends munit.FunSuite:
  test("Those should not compile") {
    val errs = typeCheckErrors("val x: Validated[LowerThan[10]] = 16")
    assertEquals(errs.map(_.message), List("Validation failed: 16 < 10"))
  }
```

If you're interested in cross-compiling your code you would be better off using munit's `compilerErrors` which for Scala 3 [uses](https://github.com/scalameta/munit/blob/7761b08fcf34396d90b22b1d086bdfd05bb733b0/munit/shared/src/main/scala-3/munit/internal/MacroCompat.scala#L38) said built-in.

Once we have an implementation for Int, it would be interesting to do the same for String.

### Iteration 3 - moving beyond Int validation

We will use the following as a motivating example:

```
val a: String Refined StartsWith["abc"] = "abcd"
```

We had to add another type parameter in addition to the predicate. To express that that `Refined[T, Predicate]` type was introduced. You can find the whole code of that [iteration](https://github.com/note/blog-examples/tree/master/build-your-own-refinement-types-in-scala3/src/main/scala/iteration3) in the accompanying repository. However, most of it is a straightforward structure not related to metaprogramming so we will jump right to the relevant bits instead.

What's interesting is how to actually implement `StartsWith` predicate at compile-time.

The first attempt might be to do the same thing that was done with Int:

```
transparent inline def checkPredString[V <: String & Singleton, E <: Pred]: Boolean =
    inline erasedValue[E] match
      case _: StartsWith[t] =>
        inline if constValue[V].startsWith(constValue[t])
        ...
```

If we try to invoke it, it will end up with such compilation error:

```
Cannot reduce `inline if` because its condition is not a constant value: "abcd".startsWith("abc")
```

The problem is that we're trying to call non-inline method `startsWith` from an inline method. Since compiler cannot reduce `startsWith` it just cannot be invoked there.

The solution to that problem is writing a simple macro:

```
transparent inline def startsWith(inline v: String, inline pred: String): Boolean =
    ${ startsWithC('v, 'pred)  }

def startsWithC(v: Expr[String], pred: Expr[String])(using Quotes): Expr[Boolean] =
    val res = v.valueOrError.startsWith(pred.valueOrError)
    Expr(res)
```

In the macro implementation (i.e. `startsWithC`) we are not limited to calling only inline methods; therefore, we can call `String.startsWith`. From my limited experience with Scala 3 macros, the tricky part is to get a value of type `T` from `Expr[T]` for non-primitive types.

We can freely call macro `startsWith` from the inline method checkPredString which completes our exercise.

#### Why `startsWith` has to be transparent

Another new Scala 3 feature used in the macro definition is modifier `transparent`. When it's used in inline method signature [2](https://msitko.pl/blog/build-your-own-refinement-types-in-scala3.html#footnote2), it allows compiler to specialize return type to a more precise type.

Getting back to our example, let's take a look at how `checkPredString` is used:

```
implicit inline def mkValString[V <: String & Singleton, E <: Pred](v: V): Refined[V, E] =
    inline if checkPredString[V, E]
    then Refined.unsafeApply(v)
    else error("Validation failed")
```

If we remove `transparent` from `checkPredString` signature the above code would fail compilation with that message:

```
Cannot reduce `inline if` because its condition is not a constant value: (true:Boolean).&&(true:Boolean):Boolean
```

Without `transparent` compiler sees return type of `checkPredString` as a Boolean which is not enough to inline the code. In contrast to that, with `transparent`, it would be a concrete type `true` or `false` depending on the validation result.

### Summary

We've ended up with a code supporting simple compile-time predicates for Int and String with very few lines of code. If you're familiar with Scala 3 metaprogramming building blocks such as inline, the code is straigforward to read. Of course, compared to real refinement types libraries there are many capabilities missing, like predicates inference or runtime lifting to refined types. This is something I explore in [mini-refined](https://github.com/note/mini-refined).

If you're interested more in Scala 3 metaprogramming capabilities, explore links in the next section.

### More resources

- [full source code](https://github.com/note/blog-examples/tree/master/build-your-own-refinement-types-in-scala3) of examples presented in this article
- [Scala 3 documentation](http://dotty.epfl.ch/docs/reference/metaprogramming/toc.html) on metaprogramming
- [talk](https://www.youtube.com/watch?v=OPBuCQRgyV4) by Josh Suereth on inline
- [mini-refined](https://github.com/note/mini-refined) - project exploring further ideas proposed in this article

1: One thing might bother you in the method signature itself:

```
implicit inline def mkVal[V <: Int & Singleton, E <: Pred](v: V): Validated[E]
```

Why do we provide value being validated both on type-level and on term-level? Wouldn't such signature suffice:

```
implicit inline def mkVal[E <: Pred](v: Int & Singleton): Validated[E]
```

Given the new signature we just need to replace all `constValue[V]` in the previous implementation with `v` and that's it, right?

The answer is _mostly yes_. It would compile indeed but whenever you call `mkVal` with a value failing validation, instead of a nice error message you would get:

```
"A literal string is expected as an argument to `compiletime.error`. Got \"Validation failed: \".+(4).+(\" < \").+(limit)
```

The issue, again, is lack of [constant folding](https://github.com/lampepfl/dotty/issues/10315) which occurs in this line:

```
error("Validation failed: " + v + " < " + limit)
```

_Reminder - in the previous version we used `constValue[V]` instead of just `v`_. Therefore, at least for now, we have to duplicate validated value on both type- and term-level. [↩](https://msitko.pl/blog/build-your-own-refinement-types-in-scala3.html#afootnote1)

2: `transparent` can be also used in [conjunction with traits](https://dotty.epfl.ch/docs/reference/other-new-features/transparent-traits.html#transparent-traits), in which case it has entirely