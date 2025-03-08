#ocaml #fp

> _In this post, I explain a trick for avoiding duplication of types between `.ml` and `.mli` files that will be familiar to anyone who's worked with Jane Street codebases._

## The problem

OCaml compilation units live a double life: one as source code (`foo.ml`) and one as header information (`foo.mli`).

![](https://www.craigfe.io/posts/the-intf-trick/two_files.svg)

This works well in encouraging abstraction, so you'll often see less type information in the `.mli` than in the `.ml`, but any types that are _not_ abstracted are duplicated. This is a big deal for functor-heavy projects, since large module types will end up being duplicated across the two files.

There's a couple of standard mitigations for this:

- **move all of your types into a single file with no corresponding `.mli`.**
    
    Each `foo.{ml,mli}` file can now alias the types and module types from a central `s.ml` or `types.ml` file. Unfortunately, all those types are now defined separately from their point-of-use, making your codebase harder to understand and less scalable.
    
- **minimise the number of module types being defined**.
    
    Since we're paying twice for each module type we define, it's natural to want to define as few of them as possible. For instance, we might avoid defining a `MAKER` type for our `Make` functor and just keep the constraints in the `.mli` file instead. Unfortunately, this hides the constraints from [Merlin](https://github.com/ocaml/merlin), so you won't discover any discrepancies until compile time:
    

```ocaml
(* --- foo.mli -------------------------------------------------------------- *)

module Make (A: Arg.S) : S with type arg = A.t

(* --- foo.ml --------------------------------------------------------------- *)

module Make (A : Arg.S) = struct

  (* We must define [type arg = A.t], but Merlin doesn't know this *)
  type arg = string

end
```

Both of these mitigations have their drawbacks. If only our `foo.ml` could refer to the module types defined in `foo.mli`. Hmm...
## The solution (?)

As with [most problems](https://en.wikipedia.org/wiki/Fundamental_theorem_of_software_engineering), we can solve this with another layer of indirection. We add a third file, named `foo_intf.ml`. This file holds types and signatures, so is like our old `foo.mli` file, but has the distinct advantage that `foo.ml` can pull types from it:

![](https://www.craigfe.io/posts/the-intf-trick/three_files.svg)

Now our types are defined in exactly one place, with no unnecessary duplication. The `foo_intf.ml` file contains all of the types required by `foo.ml` and also defines a special module type `Intf` to act as the public interface.

```ocaml
(* --- foo_intf.ml ---------------------------------------------------------- *)

(* Type definitions go here: *)

module type S = sig ... end
module type MAKER = functor (A: Arg.S) -> S with type arg = A.t
type error = [ `Msg of string | `Code of int ]

(* The interface of [foo.ml]: *)

module type Intf = sig
  type error
  (** [error] is the type of errors returned by {!S}. *)

  module type S = S
  module type MAKER = MAKER

  module Make : MAKER
end

(* --- foo.ml --------------------------------------------------------------- *)

(* Fetch module types and type definitions from the [_intf] file *)
include Foo_intf

(* Implementation here as normal *)
module Make : MAKER = functor (A : Arg.S) -> struct ... end

(* --- foo.mli -------------------------------------------------------------- *)

include Foo_intf.Intf (** @inline *)
```

There are some nice advantages to this approach:

- We've avoided duplicate definitions of `foo`'s module types _and_ kept them in the `foo*` namespace in our source tree. The code is now easier to change[1](https://www.craigfe.io/posts/the-intf-trick#fn-1) and easier to understand.

- Since we no longer have to minimise our use of module types, we can give the types of functors at the point of definition (`module Make : MAKER = ...`). This style works better with Merlin.

The `_intf` style is commonly used in Jane Street packages (c.f. [`higher_kinded`](https://github.com/janestreet/higher_kinded/tree/master/src), [`base`](https://github.com/janestreet/base/tree/master/src), [`core`](https://github.com/janestreet/core/tree/master/src)). Note that it's typically only used for files that export module types, for which this trick is most effective.

I hope you find this technique useful in making your OCaml code more concise and less frustrating to work with.

---

#### Appendix A: hiding `_intf` from Odoc

Use of the `_intf` trick is an implementation detail that (ideally) shouldn't be exposed in your documentation. At time of writing, Odoc renders all `include`s and type aliases with links to the source definition. In the case of `include`d module types, you can use the `@inline` annotation to prevent Odoc from displaying the indirection:

```ocaml
include Foo_intf.Intf (** @inline *)
```

Unfortunately: (_a_) there's no equivalent trick for plain type definitions, and (_b_) any cross-references between module types will link to the true definition. This leaves you with rendered output like the following:

```ocaml
module Make : functor (Input : Foo__.Foo_intf.INPUT) -> S
```

where `INPUT` is defined in the `Foo_intf` file but accessible to the user as `Foo.INPUT` (via an alias).

Fortunately, the [new Odoc model](https://github.com/ocaml/odoc/pull/439) solves this problem by generating links to "canonical" definitions of types, which are never taken from hidden modules (those with double underscores like `Foo__`). At time of writing, this new model hasn't yet been released.

#### Appendix B: faster file bootstrapping

An interesting side-effect of being able to reference interfaces from implementations is that you can use them to kick-start initial development on a file. If your development process begins by defining signatures, the `.ml` + `.mli` workflow requires a secondary step of "add stub implementations of everything" to sneak past the type-checker:

```ocaml
(* --- stack.mli ------------------------------------------------------------ *)

type 'a t
val empty : 'a t
val push : 'a t -> 'a -> 'a t
val pop : 'a t -> ('a t * 'a) option

(* --- stack.ml ------------------------------------------------------------- *)

type 'a t
let empty = failwith "TODO"
let push = failwith "TODO"
let pop = failwith "TODO"
```

With an `_intf` file, we can provide all of these stubs in one go:

```ocaml
(* --- stack.ml ------------------------------------------------------------- *)

include (val (failwith "TODO") : Stack_intf.Intf)
```

(I learned about this trick from a [blog post](https://blog.janestreet.com/simple-top-down-development-in-ocaml/) by Carl Eastlund.)

#### Appendix C: teaching Emacs about `_intf` files

Emacs users with `tuareg-mode` can use `tuareg-find-alternate-file` to quickly jump between corresponding `.ml` and `.mli` files. If you use this feature (as I do), you'll want it to be aware of `_intf` files. This can be done by customising the `tuareg-find-alternate-file` variable to include the correspondence `<foo>.ml` ↔ `<foo>_intf.ml`:

```elisp
;; Add support for `foo_intf.ml' ↔ `foo.ml' in tuareg-find-alternate-file
(custom-set-variables
 '(tuareg-other-file-alist
   (quote
    (("\\.mli\\'" (".ml" ".mll" ".mly"))
     ("_intf.ml\\'" (".ml"))
     ("\\.ml\\'" (".mli" "_intf.ml"))
     ("\\.mll\\'" (".mli"))
     ("\\.mly\\'" (".mli"))
     ("\\.eliomi\\'" (".eliom"))
     ("\\.eliom\\'" (".eliomi"))))))
```

If you're currently looking at some file `foo.ml`, `tuareg-find-alternate-file` will try to open `foo_intf.ml` and then `foo.mli` in that order. (If one of the two already has an open buffer, that will take priority.)

### Changelog

- **2020-06-10**: changed the recommended name of the interface `module type` from `Foo_intf.Foo` to `Foo_intf.Intf`. In the time since I originally wrote this post, I've come to dislike the duplication of the module name using the Jane Street convention: in practice, `Foo` is often quite long and subjected to later renaming.

---

1. via reducing the [repetition viscosity](https://en.wikipedia.org/wiki/Cognitive_dimensions_of_notations) of our notation for types.[↩](https://www.craigfe.io/posts/the-intf-trick#fnref-1)