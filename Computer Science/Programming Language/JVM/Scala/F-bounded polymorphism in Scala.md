#scala #fp #typesystem 

When we talk about polymorphism in programming, we're referring to the ability of an entity to take on several forms. Among the various approaches to polymorphism, _F-bounded polymorphism_ (or _F-bounded quantification_) is a particularly advanced technique in the context of object-oriented programming languages. It is characterized by its emphasis on relations between types, thereby combining the advantages of polymorphism and genericity. This concept plays a crucial role in maintaining coherent type hierarchies and promoting consistency in software development. As we explore the depths of type theory and programming, understanding F-bounded polymorphism opens doors to crafting more robust and dependable software systems.

In this article, I will start by explaining how I discovered _F-bounded polymorphism_ (somewhat by coincidence) and how it helped me in a specific case. Then, we'll take a closer look at the theory behind polymorphism with _quantification_ (basic and bounded), then _F-bounded polymorphism_, which we'll finally illustrate with a practical example. Although this concept (and polymorphism in general) exists in many languages, this article uses Scala for the examples.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-approach-by-example "Permalink")

### Approach by example

Let's start with a true story. I recently worked for a client who wanted a platform where multiple versions of information could co-exist. After developing a git-like versioning library (honourable mention to my Scala mentor who will recognize himself) that wasn't perfectly suited to the business, my team and I restarted from scratch with a new history-like approach. For reasons of confidentiality and code propriety, the code examples shown here have nothing to do with the original code, either in terms of naming or implementation (details of which are omitted to focus on _F-bounded polymorphism_ only). Let's start by defining:

```
trait Info[T] {
  def update(t: T): Info[T]

  ...
}
```

The idea is that `Info` can remember changes to an object `T`, and that for any `T` that needs to be versioned, `Info` is extended by a concrete class containing these changes field by field. An example would be:

```
case class Foo(foo: String)

case class Bar(foo: Foo, foos: List[Foo])

// definition of class Memory[T] does not matter here
case class FooInfo(fooMemory: Memory[String]) extends Info[Foo] {
  ... // implementation does not matter
}

case class BarInfo(
    fooInfo: FooInfo, 
    foosInfo: ListInfo[Foo]
) extends Info[Bar]
```

Let's now look at `ListInfo`, which, as its name suggests, represents the information of a list. To define such a class, we could imagine the following:

```
case class ListInfo[T](
    infos: List[Info[T]]
) extends Info[List[T]] {
  ...
}
```

However, as the line `foosInfo: ListInfo[Foo]` suggests, having a single parameter type `T` is not sufficient here, since the `Info[T]` type in `infos: List[Info[T]]` gives no information about the concrete class used. We can therefore modify the class as follows:

```
case class ListInfo[T, InfoType <: Info[T]](
    infos: List[InfoType]
) extends Info[List[T]]
```

We now know which `Info` type is used for the item informations. In our example, `BarInfo` becomes:

```
case class BarInfo(
    fooInfo: FooInfo, 
    foosInfo: ListInfo[Foo, FooInfo]
) extends Info[Bar]
```

Now imagine that, in `ListInfo`, we have a method for updating a particular information:

```
case class ListInfo[T, InfoType <: Info[T]](
    infos: List[InfoType]
) extends Info[List[T]] {
  def update(values: List[T]): Info[List[T]] = ???

  // imagine that update() uses this function to update the infos
  def updateInfoAtIndex(index: Int, t: T): ListInfo[T, InfoType] = {
    val updatedInfo = infos(index).update(t)
    copy(infos = infos.updated(index, updatedInfo))
  }
  ...
}
```

This code doesn't compile. Can you see the problem?

Here's the explanation. In the function `updateInfoAtIndex()`, the type of `updatedInfo` is the type of the `Info` trait's `update()` function, which, as a reminder, is:

```
def update(t: T): Info[T]
```

However, `infos` is of type `List[InfoType]`, not `List[Info[T]]`. The compiler therefore returns the following error:

```
[error]  type mismatch;
[error]  found   : updatedInfo.type (with underlying type Info[T])
[error]  required: InfoType
[error]     copy(infos = infos.updated(index, updatedInfo))
[error]                                       ^
```

We're faced here with a [type erasure](https://en.wikipedia.org/wiki/Type_erasure) of the type `InfoType` in its wider type `Info[T]`, which the compiler cannot resolve by itself. Now let's see what solutions are available to us.

#### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-naive-solution "Permalink")

#### Naive solution

The most obvious solution to our problem is type casting:

```
def updateInfoAtIndex(index: Int, t: T): ListInfo[T, InfoType] = {
  val updatedInfo: Info[T] = infos(index).update(t)
  copy(infos = infos.updated(index, updatedInfo.asInstanceOf[InfoType]))
}
```

However, this solution is sorely lacking in robustness and elegance.

#### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-robust-solution "Permalink")

#### Robust solution

Let's put the problem back at the centre of the table. Here, we lose information on the type of an object of type `InfoType <: Info[T]`, which goes back to its more general nature `Info[T]`. Intuitively, then, we might wonder whether there is a way of storing this lost information in `Info[T]` itself, at the type level. A first approach might be to use `ClassTag`'s, but we won't cover this solution in this article. Another approach is to rewrite Info as follows:

```
trait Info[T, InfoType <: Info[T, InfoType]] {
  def update(t: T): InfoType
}
```

The most remarkable thing here is the recursive definition of `Info`. Morally, `InfoType` remains a subtype of `Info`. However, with recursion, it is also a subtype of the type that "defines" it. This allows us to change the return of the `update()` function from `Info[T]` to `InfoType`. Let's look at the repercussions of this change on the `ListInfo` class:

```
case class ListInfo[T, InfoType <: Info[T, InfoType]](
    infos: List[InfoType]
) extends Info[List[T], ListInfo[T, InfoType]] {
  def update(values: List[T]): ListInfo[T, InfoType] = ???

  def updateAtIndex(index: Int, t: T): ListInfo[T, InfoType] = {
    val updatedInfo: InfoType = infos(index).update(t) // expected type
    copy(infos = infos.updated(index, infos(index).update(t)))
  }
  ...
}
```

This code now compiles. Regarding the subtypes of `Info[T]`, these need to be adapted slightly. In our example, we have

```
case class FooInfo(
    fooMemory: Memory[String]
) extends Info[Foo, FooInfo]

case class BarInfo(
    fooInfo: FooInfo, 
    foosInfo: ListInfo[Foo, FooInfo]
) extends Info[Bar, BarInfo]
```

Without really realizing it, we've just used _F-bounded polymorphism_. Let's talk about it in more detail.

## [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-polymorphism-in-type-theory "Permalink")

## Polymorphism in type theory

_F-bounded polymorphism_ is based on relationships between types. It is also nothing other than a special form of polymorphism. This concept is closely linked to that of **_type_** in programming languages. Now, you may ask: _What is a type_? Why do we need them in programming languages? A particularly appealing (and funny) answer comes from the article "On understanding types, data abstraction, and polymorphism" by L. Cardelli and P. Wegner, published in 1986:

> _A type may be viewed as a set of clothes (or a suit of armor) that protects an underlying untyped representation from arbitrary or unintended use. It provides a protective covering that hides the underlying representation and constrains the way objects may interact with other objects. In an untyped system untyped objects are naked in that the underlying representation is exposed for all to see. Violating the type system involves removing the protective set of clothing and operating directly on the naked representation._

The article concludes, among other things, that **types are sets of values**. So there are two types of language: those that are not typed, i.e. have only one type, known as _monomorphic_, and those that are typed, known as _polymorphic_. Within polymorphic languages, polymorphism can take several forms with which you're no doubt familiar: when a function works or appears to work on several different types (potentially every type), overloading, coercion, subtyping and so on. I urge you to delve into this magnificent article.

We'll now take a closer look at some common forms of polymorphism.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-basic-quantification "Permalink")

### **Basic quantification**

In type theory, _quantification_ refers to universally or existentially quantifying type variables:

- **Universal Quantification (∀)** indicates that a property holds for all permissible types. Type specifications for variables of a universally quantified type have the following form (for any type expression _σ(t)_): _p: ∀t.σ(t)_. This expresses the property that for every type _t_, _p_ has the type _σ(t)_.
    
    In Scala, _universal quantification_ is typically used via generic types, allowing functions and data structures to operate over all types _T_. For instance, a generic function might be represented as:
    

- ```
      def identity[T](x: T): T = x
    ```
    
    Here, _T_ is universally quantified: the function should work for any type _T_. Using the notation above, the identity function is written as: _∀t. t_ → _t._
    
- **Existential Quantification (∃)** denotes that there exists at least one type for which a property holds. Formally, _existential quantification_ is written as:
    
    _p:_ ∃_t_._σ(t)_. In Scala, existential types, declared using a wildcard type (placeholder syntax), signify that a type exists without specifying it:
    

```
  def printFirst(list: List[_]): Unit = println(list.headOption)
```

In this function, the type of the list elements is existentially quantified. The function knows there exists some type, but it doesn’t specify or use it explicitly.

In reality, the wildcard type (placeholder syntax) is a syntactic sugar for the formal expression of existential types in Scala, which has the form

`T forSome { Q }` where `Q` is a sequence of type declarations. Type `List[_]` can therefore be rewritten as:

- ```
      type L = List[t forSome { type t }]
    ```
    
- Note that replacing `L` with `L[_]` in the left-hand member is also valid. **Quick question for you**: how would you write the type `List[List[_]]` with this syntax? Or `List[Int]`? Or even the type representing any type? Hint: you'll find the answer in one of the sources of this article.
    

Let's end our explanation of basic _quantification_ with these few wonderful lines, which I found while wandering through the code of the [shapeless](https://github.com/milessabin/shapeless) library:

```
type ¬[T] = T => Nothing

type ∃[P[_]] = P[T] forSome { type T }
type ∀[P[_]] = ¬[∃[({ type λ[X] = ¬[P[X]]})#λ]]
```

Reading and reflecting on these lines convinces me that Scala is and always will be my favourite programming language.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-bounded-quantification "Permalink")

### **Bounded quantification**

_Bounded quantification_ is a conceptual extension of the idea of _universal_ and _existential quantification_. In essence, it is the notion of constraining the range over which a _quantification_ applies.

_Bounded quantification_ essentially introduces a restrictive layer atop basic _quantification_, enabling explicit definition of permissible type range:

- **Upper-bounded** (_T_≤_B_): ∀_T_: _T_ ≤_B_ Indicates "for all types _T_ that are subtypes of _B_. In Scala, you can express _upper-bounded quantification_ using the `<:` symbol in type parameterization:

```
def maxElement[T <: Ordered[T]](a: T, b: T): T = if (a < b) b else a
```

Here, _T_ is constrained to be a subtype of `Ordered[T]`, ensuring the elements can be ordered. More generally, if we have types _A_ and _B_, and _A_ is a subtype of _B_, it means that any value of type _A_ can also be used in a context that expects type _B_.

- **Lower-bounded** (_B_≤_T_): ∀_T_: _B_ ≤_T_ Denotes "for all types _T_ that are supertypes of _B_". Scala represents _lower-bounded quantification_ using the `>:` symbol:

```
def prependToSuperTypeList[B, T >: B](element: B, list: List[T]): List[T] = element :: list
```

Here, _T_ is a supertype of _B_, ensuring that an element of type _B_ can be prepended to a list of type _T_.

Of course, an existentially quantified parameter type can also be constrained by bounded quantification, for example:

```
def compareElements[A <: Seq[_ <: Comparable[_]]](seq1: A, seq2: A): Boolean = {
  // Comparison logic here
  true
}
```

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-f-bounded-polymorphism "Permalink")

### **F-Bounded polymorphism**

A key concept in _F-Bounded Polymorphism_ is the _F-Bound_. In _bounded quantification_, when a type A is _F-Bounded_ with respect to a type _B_, this means that instances of _A_ are linked by a particular semantic relation to those of _B_. A formalization of _F-bounded_ polymorphism appeared in 1989 in the article "F-Bounded Polymorphism for Object-Oriented Programming" by Peter Canning et al. This article presents _F-bounded polymorphism_ (or _quantification_) as a natural extension of _bounded quantification_.

In this article, we can read this definition:

> _We say that a universally quantified type is F-bounded if it has the form_
> 
> _∀t ⊆ F[t].σ_
> 
> _where F[t] is an expression, generally containing the type variable t._

Let's now break this definition into its components:

1. **∀t:** This part represents **_universal quantification_** over a type variable _t_. As stated previously, in programming languages, it means that the statement applies to all possible values of the type variable _t_.
    
2. **⊆:** This symbol represents the subtype relationship or **_upper-bounded quantification_** explained before.
    
3. **F[t]:** This is the **_type bound_** associated with the type variable _t_. It defines a set of types that _t_ must belong to. Importantly, **_F[t]_ is expressed in terms of the type variable _t_ itself**. This creates a recursive relationship, where the type bound refers to the type variable it is bounding.
    
4. **σ:** This is the actual type expression that is being quantified over and constrained by the _F-bounded_ type system. It represents the type structure that we are trying to define and apply constraints to.
    

In summary, _∀t ⊆ F[t].σ_ means that for any type _t_, the type expression _σ_ is constrained to be a subtype of the type bound _F[t]_ which is defined in terms of the type _t_ itself. In other words, if _F[t]_ is a type of the form _F[t] = {aᵢ: σᵢ[t]}_, then the condition _A ⊆ F[A]_ says that _A_ must have the methods _aᵢ_ and these methods must have arguments as specified by _σᵢ [A]_, which are defined in terms of _A_.

In Scala, probably the simplest example of _F-bounded polymorphism_ is this one:

```
trait T[U <: T[U]]
```

Please take a few seconds to admire this.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-a-symphonic-example "Permalink")

### **A symphonic example**

Let's now explain _F-bounded polymorphism_ with the analogy of musical instruments.

- **t**: A specific musical instrument (e.g., a Guitar)
    
- **F[t]**: When applied to _t_, results in an instrument constrained to harmonize with instruments of its own type
    
- **σ**: { age: InstrumentAge, produceSound: () => Sound, playInTuneWith: (F[t]) => Harmony }
    

In simpler terms, any instrument _t_ is valid only if it can play in tune with another instrument of the same type _t_ and is capable of producing sound.

Let's now express the form_∀t ⊆ F[t].σ_ in terms of Scala code:

```
enum Sound {
  case Strumming(instrumentAge: InstrumentAge)
  case Whistling(instrumentAge: InstrumentAge)
  case Harmony(sound1: Sound, sound2: Sound)
  // note that a recursive type is inherently polymorphic in nature,
  // as well as a sum type
}

import Sound._

enum InstrumentAge {
  case New,    // Crisp and clear sound.
         Old,    // Deep and resonating sound.
       Ancient // Faint and mellow sound.
}

trait Instrument[T <: Instrument[T]] {
  val age: InstrumentAge
  def produceSound(): Sound
  def playInTuneWith(instrument: T): Harmony
}

case class Guitar(age: InstrumentAge) extends Instrument[Guitar] {
  def produceSound(): Sound = Strumming(age)

  def playInTuneWith(instrument: Guitar): Harmony = Harmony(produceSound(), instrument.produceSound())
}

case class Flute(age: InstrumentAge) extends Instrument[Flute] {
  def produceSound(): Sound = Whistling(age)

  def playInTuneWith(instrument: Flute): Harmony = Harmony(produceSound(), instrument.produceSound())
}
```

Here, `Instrument` is _F-bounded_. It ensures that a `Guitar` can only play in tune with another `Guitar` and a `Flute` can only play in tune with another `Flute`. Thus the following code does not compile because the `playInTuneWith` method expects a `Piano`.

```
// compilation error
case class Piano(val age: InstrumentAge) extends Instrument[Piano] {
  def produceSound(): Sound = Whistling(age)

  def playInTuneWith(instrument: Guitar): Harmony = Harmony(produceSound(), instrument.produceSound())
}
```

**Tighter constraint with self-type annotation**

Finally, in our example, the strength of the Scala type system allows us to go even further in our constraints with self-type annotation:

```
trait Instrument[T <: Instrument[T]] { self: T =>
    ...
}
```

This definition not only asserts that `T` is a subtype of `Instrument[T]` but also guarantees that any concrete class or trait extending `Instrument[T]` is itself of type `T` . This creates a tighter constraint than the first trait definition. With this annotation, the following code does not compile anymore

```
class BlueBird extends Instrument[Guitar] {
    ...
}
```

because `BlueBird` is not of type `Guitar`.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-conclusion "Permalink")

### Conclusion

We have seen that _bounded quantification_ is a powerful tool that helps in expressing more refined and precise relationships between types.

In particular, _F-bounded polymorphism_ offers a sophisticated way of shaping type systems and ensuring logical constraints within programming languages and programs. The concept becomes simpler when seen in light of our musical analogy. It is a powerful concept in type theory that lets us constrain a type parameter based on the type itself. This recursive constraint ensures that subclasses adhere to specific type restrictions.

Using _F-bounded polymorphism_ in Scala, we achieved a type-safe way to model real-world scenarios, like musical instruments playing in tune. This ensures that mistakes like trying to tune a guitar with a flute are caught during compilation, thus eliminating potential runtime errors.

In essence, _F-bounded polymorphism_ offers an expressive and robust way to encapsulate and ensure type relations. It's like the maestro of a symphony, ensuring each instrument plays in perfect harmony, creating a melody that's both beautiful and error-free.

### [](https://brieuckaisin.hashnode.dev/f-bounded-polymorphism-in-scala#heading-sources-and-further-reading "Permalink")

### Sources and further reading

- P. Canning, W. Cook, W. Hill and W. Olthof. F-Bounded Polymorphism for Object-Oriented Programming. _Proceedings of the fourth international conference on Functional programming languages and computer architecture_. 1989. [https://www.cs.utexas.edu/~wcook/papers/FBound89/CookFBound89.pdf](https://www.cs.utexas.edu/~wcook/papers/FBound89/CookFBound89.pdf)
    
- L. Cardelli and P. Wegner. On understanding types, data abstraction, and polymorphism. _Computing Surveys_, 17(4):471–522, 1986. [http://lucacardelli.name/papers/onunderstanding.a4.pdf](http://lucacardelli.name/papers/onunderstanding.a4.pdf)
    
- The Scala 2.11 specification of existential types. [https://www.scala-lang.org/files/archive/spec/2.11/03-types.html#existential-types](https://www.scala-lang.org/files/archive/spec/2.11/03-types.html#existential-types)
    
- The Curiously Recurring Template Pattern (CRTP) in C++. See for example [https://en.cppreference.com/w/cpp/language/crtp](https://en.cppreference.com/w/cpp/language/crtp).