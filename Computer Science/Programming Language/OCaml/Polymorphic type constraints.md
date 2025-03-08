#ocaml #fp #typesystem

> _In this post, I explain a common mistake when writing constraints of polymorphic functions in OCaml programs, then show how to correct it._

## Not-so-polymorphic constraints

One of the earliest lessons of any functional programming tutorial is how to write polymorphic functions and their signatures:

```ocaml
val id  : 'a -> 'a
val fst : ('a * 'b) -> 'a
val map : ('a -> 'b) -> 'a list -> 'b list
```

A typical explanation of these type signatures goes along the lines of:

> _Types of the form `'a`, `'b`, ..., known as **type variables**, stand for an unknown type. They allow us to describe functions that work uniformly over many possible input types. This is known as "parametric polymorphism"._
> 
> — Hypothetical education resource[1](https://www.craigfe.io/posts/polymorphic-type-constraints#fn-1)

As is often the case with introductory explanations, this is _just_ specific enough to be technically correct without introducing too many new concepts, letting us hurry on to demonstrating useful examples before the student gets bored. Unfortunately, we've laid a trap: when our reader learns about type constraints, they naturally try to combine these two "intuitive" features and get bitten:

```ocaml
ᐅ let id1 : 'a -> 'a = (fun x -> x) ;;        (* Accepted. So far so good... *)
  val id1 : 'a -> 'a = <fun>

ᐅ let id2 : 'a -> 'a = (fun x -> x + 1) ;;    (* Also accepted. Uh oh... *)
  val id2 : int -> int = <fun>
```

In this case, the student finds that `'a -> 'a` is a valid constraint for a function of type `int -> int`, and their mental model is broken almost immediately. It's quite natural to expect `id2` to be rejected as a non-polymorphic function, particularly given our vague explanation of what `'a` actually means.

Our hypothetical student's mistake stems from the fact that type variables in signatures are implicitly _universally-quantified_ – that is, they stand for _all_ types – whereas type variables in constraints are not. To understand what this means, let's try to pin down a more precise idea of what type variables _are_. If you're already  ~~indoctrinated~~  comfortable with type variables, you may wish to [cut to the chase](https://www.craigfe.io/posts/polymorphic-type-constraints#true-polymorphic-constraints).

## What is a type variable anyway?

Type variables in constraints are referred to as being "unbound" (or "free"), meaning that they stand for _some_ type that is not yet known to the type-checker: they are placeholders that can later be filled by a particular type. Without going into [the details](http://dev.stephendiehl.com/fun/006_hindley_milner.html), these placeholders are gradually determined as the type-checker resolves constraints. For instance, in our `id2` example, the type-checker decides that `'a` equals `int` by first reconciling the user-supplied constraint `'a -> 'a` with the constraint `int -> int` that it inferred from the implementation.

To a theorist (or type-system developer), who regularly has to worry about types that are not yet fully known, the notion of a "placeholder" is a sensible default meaning of an unbound type variable. Such people also tend to use explicit syntax to disambiguate the alternative case, type variables that _are_ bound:

> `∀ a. a -> a` (read as: _"For all `a`, `a -> a`"_)

We call "`∀ a`" a _universal quantifier_ because it introduces a variable `a`, bound inside the quantifier, that can stand for any type in the universe of OCaml types. It's this flavour of type variable that enables parametric polymorphism and – although the OCaml syntax often tries to hide it from you – these quantifiers exist everywhere in your programs. As I already mentioned, all unbound variables in _signatures_ are implicitly quantified in this way:

> `val length : 'a list -> int`
> 
> _... secretly means ..._
> 
> `val length : ∀ a. a list -> int`

On the implementation side of `length`, the compiler will check to see if there are any placeholder variables left after type-checking the definition and wrap them in universal quantifiers (if it's sure that it's safe to do so[2](https://www.craigfe.io/posts/polymorphic-type-constraints#fn-2)). When this happens, we say that those type variables have been _generalised_. Once `length` has been given its polymorphic type, the user gets to pick a specific type `a` at each call-site by passing it a list of any element type they want. This idea of choosing the instantiation of `a` at each call-site is what is "parametric" about "parametric polymorphism".

Taking a step back, we can now see what went wrong with our hypothetical introduction to type variables above: it led our student to think of all type variables as being implicitly universally-quantified, when this is not true in constraints[3](https://www.craigfe.io/posts/polymorphic-type-constraints#fn-3). So, given that we can't rely on implicit generalisation in constraints, what _can_ we do to declare that our code is polymorphic within the implementation itself?

## True polymorphic constraints

The punchline is that OCaml actually _does_ have syntax for expressing polymorphic constraints – and it even involves an explicit quantifier – but sadly it's not often taught to beginners:

```ocaml
let id : 'a. 'a -> 'a = (fun x -> x + 1)
```

The syntax `'a. 'a -> 'a` denotes an [explicitly-polymorphic type](https://caml.inria.fr/pub/docs/manual-ocaml/types.html#poly-typexpr), where `'a.` corresponds directly with the `∀ a.` quantifier we've been using so far. Applying it here gives us a satisfyingly readable error message:

```text
Error: This definition has type int -> int which is less general than
         'a. 'a -> 'a
```

The caveat of polymorphic constraints is that we can only apply them directly to `let`-bindings, not to function bodies or other forms of expression:

```ocaml
let panic : 'a. unit -> 'a = (fun () -> raise Crisis)  (* Works fine... *)

let panic () : 'a. 'a = raise Crisis                   (* Uh oh... *)
(*               ^
 *  Error: Syntax error  *)
```

This somewhat unhelpful error message arises because OCaml will never infer a polymorphic type for a value that is not `let`-bound. Trying to make your type inference algorithm cleverer than this quickly runs into certain [undecidable problems](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/putting.pdf); the parser knows that the type-checker is afraid of undecidable problems, and so rejects the program straight away[4](https://www.craigfe.io/posts/polymorphic-type-constraints#fn-4).

In spite of their limitations, explicitly-polymorphic type constraints are a great way to express polymorphic intent in your OCaml programs, either as internal documentation or to have more productive conversations with the type-checker when debugging. I recommend using them frequently and teaching them to beginners as soon as possible.

At this point, if you suffered through my explanation of type variables in the previous section, you may be thinking the following:

> _"If introducing type variables properly requires so many paragraphs of jargon, we **shouldn't** burden beginners with the details right away."_
> 
> — Straw-man argument

Personally, I find that introducing these terms early on in the learning process is easily worthwhile in avoiding early roadblocks, but that discussion can wait for another time. In the spirit of functional programming for the masses, let's summarise with a less jargon-heavy attempt at redrafting our hypothetical education resource:

> _Types of the form `'a`, `'b`, ..., known as type variables, are placeholders for an undetermined type. When bound by for-all quantifiers (of the form `'a.`), they can be used to describe values that can take on many possible types. For instance, we can write the type of `(fun x -> x)` as `'a. 'a -> 'a`, meaning:_
> 
> > _"For any type `'a`, this function can take on type `'a -> 'a`."_
> 
> _The OCaml type-checker will infer polymorphic types wherever it is safe, but we can also explicitly specify a polymorphic type for a `let`-binding:_
> 
> ```ocaml
> ᐅ let fst : 'a 'b. ('a * 'b) -> 'a = (* "For all ['a] and ['b], ..." *)
>     fun (x, _) -> x ;;
> 
> val fst : 'a * 'b -> 'a = <fun>
> ```
> 
> _Note that all type variables in signatures are implicitly universally-quantified: it's not necessary (or even possible) to write `'a 'b.` before the type._

The explanation is undeniably still longer and more technical than the one we started with, but crucially it uses the extra space to give the reader a clue as to how to debug their polymorphic functions.

The story doesn't end here. We haven't discussed [existential quantifiers](https://caml.inria.fr/pub/docs/manual-ocaml/gadts.html), the other type of type variable binding; or [polymorphic recursion](https://caml.inria.fr/pub/docs/manual-ocaml/polymorphism.html#s:polymorphic-recursion), where polymorphic annotations become compulsory; or [locally-abstract types](https://caml.inria.fr/pub/docs/manual-ocaml/locallyabstract.html), which offer other useful syntaxes for constraining your OCaml programs to be polymorphic. These will all have to wait for future posts. For now, thanks for reading!

---

1. Very similar equivalents of this explanation exist in [Real World OCaml](http://dev.realworldocaml.org/guided-tour.html), [Cornell's OCaml course](https://www.cs.cornell.edu/courses/cs3110/2020sp/textbook/), and [Cambridge's OCaml course](https://www.cl.cam.ac.uk/teaching/1920/FoundsCS/focs-201920-v1.1.pdf). Type variables are variously described as representing _"any type"_, _"an unknown type"_ or _"a generic type"_; explanations that are all as different as they are vague.[↩](https://www.craigfe.io/posts/polymorphic-type-constraints#fnref-1)
    
2. The most famous example of a type variable that is unsafe to generalise is one that has been captured in mutable state:
    
    ```ocaml
    ᐅ let state = ref [] ;;
    ᐅ let sneaky_id x = (state := x :: !state); x ;;
    
    val sneaky_id : '_weak1 -> '_weak1 = <fun>
    ```
    
    In this case, it's not possible to give `sneaky_id` the type `∀ a. a -> a` because different choices of the type `a` are not independent: passing a string to `sneaky_id`, followed by an integer, would build a list containing both strings and integers, violating type safety. Instead, `sneaky_id` is given a type containing a "[weak type variable](https://caml.inria.fr/pub/docs/manual-ocaml/polymorphism.html#ss:weak-types)" which represents a single, unknown type. This meaning of type variables should be familiar to you; it's exactly the same as the "unbound" type variables we've been discussing!
    
    In general, it's not easy to decide if it's safe to generalise a particular type variable. OCaml makes a quick under-approximation called the [(relaxed) value restriction](https://caml.inria.fr/pub/docs/manual-ocaml/polymorphism.html#ss:valuerestriction).
    
    [↩](https://www.craigfe.io/posts/polymorphic-type-constraints#fnref-2)
    
3. As an aside, there's no profound reason why constraints must behave differently with respect to implicit quantification. Both SML and Haskell choose to generalise variables in constraints:
    
    ```ocaml
    val id1 : 'a -> 'a = (fn x => x + 1);
    (*  Error: pattern and expression in val dec do not agree
    
    pattern:    'a -> 'a
    expression: 'Z[INT] -> 'Z[INT] *)
    ```
    
    > (Note: `val f : t = e` in SML is analogous to `let f : t = e` in OCaml.)
    
    I suspect that constraints having the same quantification behaviour as signatures is more intuitive, at least for simple examples. In complex cases, the exact point at which type variables are implicitly quantified can be surprising, and so SML '97 provides an explicit quantification syntax for taking control of this behaviour. See [the SML/NJ guide (§ 1.1.3)](https://www.smlnj.org/doc/Conversion/types.html#Explicit) for much more detail.
    
    The advantage of OCaml's approach is that it enables constraining _subcomponents_ of types without needing to specify the entire thing (as in `(1, x) : (int * _)`), which can be useful when quickly constraining types as a sanity check or for code clarity. As far as I'm aware, SML has no equivalent feature.
    
    [↩](https://www.craigfe.io/posts/polymorphic-type-constraints#fnref-3)
    
4. This limitation of the type-checker can artificially limit the polymorphism that can be extracted from your programs. If you want to take polymorphism to its limits – as God intended – it's sometimes necessary to exploit another point where explicitly-polymorphic types can appear: [record and object fields](https://caml.inria.fr/pub/docs/manual-ocaml/polymorphism.html#s%3Ahigher-rank-poly).[↩](https://www.craigfe.io/posts/polymorphic-type-constraints#fnref-4)