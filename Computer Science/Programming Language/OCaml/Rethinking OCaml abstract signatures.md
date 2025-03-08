#ocaml #fp #typesystem

Abstract module types are one of the less understood features of the OCaml module system. They have been one of the obstacles in the on-going effort to specify, and eventually redesign, the module system.

In this blog post, I (Clément Blaudeau) present an explanation of what are those abstract module types, and propose a slightly restricted version that might be easier to understand and specify while remaining pretty expressive.

For the past 2 years, I’ve been working on building a new specification for the OCaml module system based on Fω. The goal, besides the theoretical interest, is to eventually redo (the module part of) the typechecker with this approach, which would have several benefits:

- fix some soundness issues and edge-cases that have appeared and built up over the years, due to unforeseen interactions between features
- simplify the (notoriously hard) code of the typechecker by removing ad-hoc techniques and hacks (such as the _strengthening_ or the treatment of _aliases_ for instance)
- provide a clean base to add new and awaited features. Notably, transparent ascription and modular implicits are proposals for OCaml modules stalled by the lack of specification of the module system.

Yet, a key aspect in OCaml development culture is to ensure backward compatibility. Therefore, the new Fω approach I’ve been building should not only subsumes the current typechecker in normal use cases, but actually support _all_ of the features of the module system. For long, _abstract signatures_ (also called _abstract module types_) were believed to be, at least, _problematic_ for Fω. Hopefully, we found out that a slightly restricted version of the feature was encodable in Fω, and, in passing, made the semantics of abstract signatures much simpler. Thus, only one question remains: does this restricted form actually covers all use cases, i.e., is the restriction backward compatible ?

Here, we aim at presenting the current state of abstract signatures and our proposed simplification purely from an **OCaml user point of view**, not from the theoretical one. We welcome any feedback, specifically, use cases or potential use cases that significantly differ from our examples.

We start by introducing abstract signatures through examples. Then, we present the current state of abstract signatures in OCaml: we explain the syntactic approach and the issues associated with it. We argue that it has surprising behaviors and, in its current _unrestricted_ form, it is actually _too powerful for its own good_. Then, we propose a restriction to make the system _predicative_ which, by decreasing its expressiveness, actually makes it more usable. (Our actual proposal is given in 3.3). We finish by other aspect related to usability (syntax, inference).

#### 1. What are abstract signatures ?

The art of modularity is all about controlling abstraction and interfaces. ML languages offer this control via a _module system_, which contains a _signature_ language to describe interfaces. Signatures contain _declarations_ (fields): values `val x : t`, types `type t = int`, modules `module X : S`, and module types `module type T = S`. Type and module type declarations can also be abstract `type t`, `module type T`, which serves both to hide implementation details via _sealing_ and to have polymorphic interfaces, using _functors_.

Here, we focus on the construct `module type T`, called _abstract module type_ or _abstract signature_. We start with examples adapted from [this forum discussion](https://discuss.ocaml.org/t/what-are-abstract-module-types-useful-for/10121/3).

##### 1.1 Module-level sealing

Let’s consider the following scenario. Two modules providing an implementation of UDP (`UDP1` and `UDP2`) are developed with different design trade-offs. They both implement a signature with basic send and receive operations. Then, functors are added as layers on top: taking a udp library as input, they return another udp library as an output.

- `Reliable` adds sequence numbers to the packets and re-sends missing packets;
- `CongestionControl` tracks the rate of missing packets to adapt the throughput to network congestion situations;
- `Encryption` encrypts the content of all messages.

A project might need different combinations of the basic libraries and functors, while requiring that **all** combinations use encryption. To enforce this, the solution is to use the module-level _sealing_ of abstract signatures. In practice, the signature of the whole library containing implementations and functors `UDPLib` (typically, its `.mli` file) is rewritten to abstract all interfaces except for the output of the `Encryption` functor.

module type UDPLib = sig
  module type UNSAFE

  module UDP1 : UNSAFE
  module UDP2 : UNSAFE

  module Reliable : UNSAFE -> UNSAFE
  module CongestionControl : UNSAFE -> UNSAFE

  module Encryption : UNSAFE ->
    sig val send : string -> unit (* ... *) end
end

Just as type abstraction, signature abstraction can be used to enforce certain code patterns: users of `UDPLib` will only be able to use the content of modules after calling the `Encryption` functor, and yet they have the freedom to choose between different implementations and features:

module UDPKeyHandshake = Encryption(Reliable(UDP1))
module UDPVideoStream  = Encryption(CongestionControl(UDP2))
(* etc *)

##### 1.2 Module-level polymorphism

Another use is to introduce polymorphism at the module level. Just as polymorphic functions can be used to factor code, module-level polymorphic functors can be used to factor module expressions. If a code happens to often feature functor applications of the form `Hashtbl.Make(F(X))` or `Set.Make(F(X))`, one can define the `MakeApply` functor as follows:

(* Factorizing common expressions *)
module type Type = sig module type T end
module MakeApply
  (A:Type) (X: A.T)
  (B:Type) (F: A.T -> B.T)
  (C:Type) (H: sig module Make : B.T -> C.T end) = H.Make(F(X))

Downstream the code is rewritten into `MakeApply(...)(X)(...)(F)(...)(Set)` or `MakeApply(...)(X)(...)(F)(...)(Hashtbl)` Right now, the verbosity of such example would probably be a deal-breaker. We address this aspect at the end. Ignoring the verbosity, this can be useful for maintenance: by channeling all applications through `MakeApply`, only one place needs to be updated if the arity or order of arguments is changed. Similarly, if several functors expect a _constant argument_ containing – for instance – global variables, a `ApplyGv` functor can be defined to always provide the right second argument, which can even latter be hidden away to the user of `ApplyGv`:

(* Constant argument *)
module Gv : GlobalVars
module ApplyGv (Y : sig module type A module type B end)
    (F : Y.A -> GlobalVars -> Y.B)(X : Y.A) = F(X)(Gv)

Downstream, code featuring `F(X)(GlobalVars)` is rewritten into `ApplyGv(...)(F)(X)` Then, the programmer can hide the `GlobalVars` module while letting users use `ApplyGv`, ensuring that global variables are not modified in uncontrolled ways by certain part of the program.

Finally, polymorphism can also be used by a developer to prevent unwanted dependencies on implementation details. If the body of a functor uses an argument with a submodule `X`, but actually does not depend on the content of `S`, abstracting it is a “good practice”.

module F (Arg : sig ... module X : S ... end) =
  ... (* polymorphism is not enforced *)

module F' (Y: sig module type S end)
    (Arg : sig ... module X : Y.S ... end ) =
  ... (* polymorphism is enforced *)

##### 1.3 So it’s just normal use cases of abstraction ?

Fundamentally, these example are not surprising for developers that are used to rely on abstraction to protect invariants and factor code. Their specificity lies in the fact that there are at the module level, and therefore require projects with a certain size and a strong emphasis on modularity to be justified.

#### 2. OCaml’ abstract signatures, an incidental feature ?

The challenge for understanding (and implementing) abstract signatures lies more in the meaning of the module-level _polymorphism_ that they offer than the module level sealing, the latter being pretty straightforward. More specifically, the crux lies in the meaning of the _instantiation_ of an abstract signature variable `A` by some other signature `S`, that happens when a polymorphic functor is applied. OCaml follows an _unrestricted syntactical approach_: `A` can be instantiated by any (well-formed) signature `S`. During instantiation, all occurrences of `A` are just replaced by `S` ; finally, the resulting signature is **re-interpreted**—as if it were written _as is_ by the user.

However, this syntactical rewriting interferes with the _variant_ interpretation of signatures, which can lead to surprising behaviors. We discuss this aspect first. The _unrestricted_ aspect leads to the (infamous) _Type : Type_ issue which has some theoretical consequences. We finish this section by mentioning other—more technical—issues.

##### 2.1 Syntactical rewriting

The first key issue of this approach comes from the fact that signatures in OCaml have a variant interpretation: abstract fields (1) have a different meaning (sealing or polymorphism) depending on whether they occur in positive or negative positions, and (2) abstract fields open new scopes, i.e. duplicating an abstract type field introduces two different abstract types. Overall, OCaml signatures can be thought of as having _implicit quantifiers_: using a signature in positive or negative position changes its implicit quantifiers (from existential to universal) while duplicating a signature duplicates the quantifiers (and therefore introduces new incompatible abstract types).

Therefore, when instantiating an abstract signature with a signature that has abstract fields, the user must be aware of this, and mentally infer the meaning of the resulting signature. To illustrate how it can be confusing, let’s revisit the first motivating example and let’s assume that the developer actually want to expose part of the interface of the raw `UDP` libraries. One might be tempted to instantiate `UNSAFE` with something along the following lines:

module type UDPLib_expose = sig
  include UDPLib with module type UNSAFE =
    sig
      module type CORE_UNSAFE
      module Unsafe : CORE_UNSAFE (* this part remains abstract *)
      module Safe : sig ... end (* this part is exposed *)
    end
end

This returns :

module type UDPLib_expose =  sig
  module type UNSAFE =
    sig
      module type CORE_UNSAFE
      module Unsafe : CORE_UNSAFE
      module Safe : sig ... end
    end
  module UDP1 : UNSAFE
  module UDP2 : UNSAFE
  module Reliable : UNSAFE -> UNSAFE
  module CongestionControl : UNSAFE -> UNSAFE
  module Encryption : UNSAFE -> sig val send : string -> unit (* ... *) end
end

However, the syntactical rewriting and reinterpretation of this signature in the negative positions produces a counter-intuitive result. For instance, if we expand the signature of the argument for the functor `Reliable` (for instance) we see:

module Reliable :
sig
  module type CORE_UNSAFE
  module Unsafe : CORE_UNSAFE
  module Safe : sig ... end
end -> UNSAFE

This means that the functor actually has to be _polymorphic_ in the underlying implementation of `CORE_UNSAFE`, rather than using the internal details, which has the **opposite meaning** as before. If the user wants to hide a _shared_ unsafe core, accessible to the functor when they were defined and then abstracted away, the following pattern may be used instead:

module type UDPLib_expose' = sig
  module type CORE_UNSAFE
  include UDPLib with module type UNSAFE = sig
    module type CORE_UNSAFE = CORE_UNSAFE
    module Unsafe : CORE_UNSAFE
    module Safe : sig ... end
  end
end

Doing so, the instantiated signature does not contain abstract fields and therefore its variant reinterpretation will not introduce unwanted polymorphism. This observation is at the core of the proposal of this post.

##### 2.2 `Type : Type` and impredicativity

Abstract module types are impredicative: a signature containing an abstract signature can be instantiated by itself. One can trick the subtyping algorithm into an infinite loop of instantiating an abstract signature by itself, as shown by [Andreas Rosseberg](https://sympa.inria.fr/sympa/arc/caml-list/1999-07/msg00027.html), adapting an example from [Harper and Lillibridge (POPL ’94)](https://doi.org/10.1145/174675.176927). This also allows type-checking of (non-terminating) programs with an absurd type, as shown by the encoding of the Girard’s paradox done by [Leo White](https://github.com/lpw25/girards-paradox/tree/master).

##### 2.3 Other issues

The current implementation of the typechecker does not handle abstract signatures correctly in some scenarios. It’s unclear if they are just bugs or pose theoretical challenges.

###### 2.3.1 Invalid module aliases

Inside a functor, module aliases are disallowed between the parameter and the body (for soundness reasons, due to coercive subtyping). However, this check can be bypassed by using an abstract signature that is then instantiated with an alias. If we try to use it to produce a functor that exports its argument as an alias, the typechecker crashes. This is discussed in [#11441](https://github.com/ocaml/ocaml/issues/11441)

(* crashes the typechecker in current OCaml *)
module F (Type : sig module type T end)(Y : Type.T) = Y

module Crash (Y : sig end) =
  F(struct module type T = sig module X = Y end end)

###### 2.3.2 Loss of applicativity

The use of abstract signatures clashes with applicativity of functors, as discussed [in #12204](https://github.com/ocaml/ocaml/issues/12204).

###### 2.3.3 Invalid signatures and avoidance

Another known issue is that the typechecker can abstract a signature when it contains unreachable type fields (types pointing to anonymous modules). This can lead to the production of _invalid signatures_ : signatures that are refused by the typechecker when re-entered back in.

module F (Y: sig type t end) =
  struct
    module type A = sig
      type t = Y.t (* this will force the abstraction of all of A *)
      type u
    end
    module X : A = struct type t = Y.t type u = int end
    type u = X.u
  end

module Test = F(struct type t end)

(* returns *)
module Test : sig module type A module X : A type u = X.u end

Here, the type field `type u = X.u` is invalid as `X` has an abstract signature (and therefore, no fields).

#### 3. A solution: simple abstract signatures

In this section we explore solutions for fixing the issues of the current approach. The core criticism we make of the OCaml approach is that it is actually _too expressive for its own good_. Abstract signatures are _impredicative_: they can be instantiated by themselves. Having impredicative instantiation with variant reinterpretation is hard to track for the user and interacts in very subtle ways with other features of the module system, slowing down its development—and breaking its theoretical properties. To address this, we take the opposite stance and propose to make the system actually _predicative_: we restrict the set of signatures that can be used to instantiate an abstract signature. This also indirectly addresses the complexity of the variant reinterpretation.

We start with the simplest solution where instantiation of abstract signatures is restricted to signatures containing no abstract fields. Then, we propose to relax this restriction and allow for signatures that contain abstract _type_ fields (but no abstract module types), which we call _simple_ signatures. This will requires us to briefly discuss the need for module-level sharing.

In this section we focus on the theoretical aspects, but present them informally with examples. The practical aspects, notably syntax and inference, are discussed in the next section.

##### 3.1 No abstraction

One might wonder why abstract types and abstract signatures syntactically resembles one another and yet, the latter is much more complex than the former. The key lies in the fact that abstract types can only be instantiated by _concrete_ type expressions, without free variables. Informally, this:

sig
  type t
  val x : t
  val f : t -> t
end with type t = (int * 'a)

is not allowed, notably because (1) the scope of the abstract type variable `'a` is unclear, (2) values of type `t`, like `x`, would be ill-typed.

Therefore, a first solution is to **require abstract signatures to be instantiated only by concrete signatures, i.e. signatures with no abstract fields** (neither types nor module types). This circumvents the clash between the rewriting and variant reinterpretation of abstract fields (by disallowing them).

This is simple and sound but prevents some valid uses of abstract types: in the first example, `UNSAFE` could not be instantiated with abstract type fields, forcing `UDP1` and `UDP2` to have the same type definitions.

##### 3.2 The issue of module-level sharing

If we want to relax the no-abstraction proposal, some abstract fields will be allowed when instantiating signatures. Then, the question of what sharing (i.e., type equalities) should be kept _between different occurrences of the abstract fields_ arises.

In OCaml signatures, sharing between two modules is usually expressed _at the core-level_ by rewriting the fields of the signature of the second module to refer to their counterpart in the first one. This cannot be done with abstract signatures, as they have no fields. Instead, the language needs module-level sharing, which in OCaml is very restricted. Indeed, it provides a form of module aliases (only for submodules, not at the top-level of a signature), but aliasing between a functor body and its parameter is not allowed—while it is typically the use-case for abstract signatures in polymorphic functors. Consider the following code:

(* Code *)
module F1 (Y: sig module type A module X : A end) = Y.X
module F2 (Y: sig module type A module X : A end) = (Y.X : Y.A)

Currently, the typechecker cannot distinguish between the two and returns the same signature, while we would expect the first one to keep the sharing between the parameter and the body.

(* Currently, both are given the same type: *)
module F1 (Y: sig module type A module X : A end) : A
module F2 (Y: sig module type A module X : A end) : A

As an example, we can consider the argument for the functors:

module Y = struct
  module type A = sig type t end
  module X = struct type t = int end
end

module Test1 = F1(Y)
module Test2 = F2(Y)

This returns :

module Test1 : sig type t end
module Test2 : sig type t end

While we would expect :

module Test1 : sig type t = int end
module Test2 : sig type t end

Two possible extensions would help tackle this issue.

###### Lazy strengthening

A recently proposed experimentation, named _lazy strengthening_, extends the signature language with an operator `S with P`, where `S` is a signature and `P` a module path. It is interpreted as `S` strengthened by `P`, i.e. `S` in which all abstract fields are rewritten to point to their counterpart in `P`. Initially considered for performance reasons, it would allow for tracking of type equalities when using abstract signatures.

(* Lazy strengthening would keep type equalities: *)
module F1 (Y: sig module type A module X : A end) = Y.A with Y.X

###### Transparent ascription

A more involved solution is the use of an extension of aliasing called _transparent ascription_, where both the alias _and_ the signature are stored in the signature. The signature language would be extended with an operator `(= P < S)`. The technical implications of this choice are beyond the scope of this discussion.

(* Transparent ascription would keep module equalities: *)
module F1 (Y: sig module type A module X : A end) : (= Y.X < Y.A)

##### 3.3 Our proposal : _simple_ abstract signatures

Maintaining a predicative approach, we propose to **restrict instantiation only by _simple_ signatures, i.e., signatures that may contain abstract type fields, but no abstract module types**. This reintroduces the need to express module-level sharing and the mental gymnastic of variant re-interpretation of abstract type fields. However, it guarantees that all modules sharing the same abstract signature will also share the same structure (same fields) after instantiation, and can only differ in their type fields. We believe this makes for a good compromise.

###### Expressivity (and prenex-form)

One might wonder how restrictive is this proposal. Specifically, if we consider a simple polymorphic functor as:

module Apply (Y : sig module type A end) (F : Y.A -> Y.A)(X : Y.A) = F(X)

The following partial application would be rejected:

(* Rejected as A would be instantiated by `sig module type B module X : B -> B end` *)
module Apply' = Apply(struct module type A = sig module type B module X : B -> B end end)

However, this could be circumvented by eta-expanding, thus expliciting module type parameters, and instantiating only a simple signature:

(* Accepted as A is instantiated by a signature with no abstract fields *)
module Apply'' = functor (Y:sig module type B end) ->
  Apply(struct module type A = sig module type B = Y.B module X : B -> B end end)

###### Higher-abstraction ?

Concrete and simple signatures can be seen as the first two levels of the predicative approach for types declarations. There are no more levels for type declarations, as types cannot be _partially abstract_ (see 3.1). Could it be useful to add even more expressivity and authorize instantiation by a signature containing again an abstract module type field (which would need to be restricted with a level system like _universes_)? We have found no example where this was useful. Besides, it would add a great layer of complexity.

#### 4. Other practical aspects

###### Syntax

A key aspect of abstract module types that reduces their usability is the verbosity of the syntax. Rather than having to pass signature as part of a module argument to a polymorphic functor, using a separate notation for module type parameters could be more concise. In practice, abstract signature arguments could be indicated by using brackets instead of parenthesis, and interleaved with normal module arguments, as in this example:

(* At definition *)
module MakeApply
    [A] (X:A)
    [B] (F: A -> B)
    [C] (H : sig module Make : B -> C end)
  = H.Make(F(X))

module ApplyGv
    [A] [B] (F:A -> GlobalVars -> B) (X:A)
  = F(X)(Gv)

(* At the call site *)
module M1 = MakeApply
    [T] (X)
    [Hashtbl.HashedType] (F)
    [Hashtbl.S] (Hashtbl)

module M2 = ApplyGv [A] [B] (F) (X)

Technically, this is not just syntactic sugar for anonymous parameters due to the fact that OCaml relies on names for applicativity of functors.

###### Inference

Following up on the previous point, usability of abstract signatures could even be improved with some form of inference at call sites. Further work is needed to understand to what extend this could be done.

#### Conclusion

We have presented the feature of _abstract signatures_ in OCaml. After showing use cases via examples, we explained the issues associated with the unrestricted syntactical approach. Then, we propose a new specification: _simple abstract signatures_. In addition to making the behavior of abstract signatures much more predictable for the user, this approach can be fully formalized by translation into Fω (extended with predicative kinds).

As stated above, our goal here was both to sum up the current state and our proposal, but also to gather feedback from users or potential users. In particular, we want to see if it can indeed cover all use cases, and if we missed other usability problems.