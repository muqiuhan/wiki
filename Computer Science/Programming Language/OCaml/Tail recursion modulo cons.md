#ocaml #fp #typesystem

If the last action of a function ff is to call another function gg, the language run-time doesn't need to keep ff's stack frame around when calling gg:

```ocaml
let f n =
  Printf.printf "Hello, World!";
  g (n + 1)
```

Instead, the run-time may `re-purpose' ff's stack frame for gg, saving space and time in stack (de)allocations. This optimisation, known as _tail call elimination_, is useful in many language paradigms. It is useful in _functional_ programming languages for which recursion is the idiomatic way to repeat actions.

Unfortunately, many standard uses of recursion are _not_ tail-call optimisable:

```ocaml
let rec map f = function
  | [] -> []
  | x :: xs ->
     let y = f x in
     y :: map f xs
```

The x::xsx::xs case proceeds as follows:

1. Compute y:=f(x)y:=f(x);
2. Recursively compute t1:=map f xst1​:=map f xs;
3. Allocate t2:=y::t1t2​:=y::t1​ on the heap;
4. Return t2t2​.

Step 3 prevents the tail-call optimisation: the runtime must build the list node _after_ computing the tail with `map f xs`. Our `map` function is _almost_ tail-recursive: if not for the data constructor `(::)`, it would be. We call such functions 'tail recursive _modulo cons_'. There are two ways to make `map` fully tail-recursive:

- We could build the result list in reverse order, then reverse it in one pass at the end. This is not ideal since our intermediate list requires time to build and creates work for the garbage collector.
    
- We could change the `list` type to allow us to build the list node _first_, and later fill in the correct tail. In OCaml, this needs a `ref` indirection.
    

Let's try the latter approach. We introduce a `ref` and pass the 'tail to be filled in later' as an explicit argument:

```ocaml
type 'a mutable_list = (::) of 'a * 'a mutable_list ref | []

let rec map f xs =
    let rec inner res = function
     | [] -> ()
     | x :: xs ->
        let y = f x in
        (* create an 'incomplete' list node *)
        let tail = ref [] in
        res := y :: tail;
        inner tail !xs (* tail call! *)
    in
    let res = ref [] in
    inner res xs; !res
```

Putting 'incomplete' values into the heap _as a user_ requires changing the `list` type to contain references, but the OCaml runtime doesn't have the same restriction! It can choose to modify 'immutable' heap contents if it wants, allowing our original `map` to be compiled to take O(1)O(1)-space and not generate any garbage!

The transformation given above can be applied whenever a function is 'tail recursive modulo cons': whenever the only actions after the last function call are heap allocations. The OCaml compiler doesn't _yet_ make this optimisation, but it could! There are interesting details to be fixed, such as what happens when a garbage collection happens in the middle of the TRMC recursion.[1](https://www.craigfe.io/posts/tail-recursion-modulo-cons#fn-1)

  

> Many thanks to [@Splingush](https://discord.com/users/327286755562618891/) for corrections to this post.

---

1. [http://gallium.inria.fr/seminaires/transparents/20141027.Frederic.Bour.pdf](http://gallium.inria.fr/seminaires/transparents/20141027.Frederic.Bour.pdf)[↩](https://www.craigfe.io/posts/tail-recursion-modulo-cons#fnref-1)