#ocaml #fp #typesystem

In OCaml, we can ignore the return value of a side-effecting computation by either naming it `_` or using `Stdlib.ignore`. These are commonly used to test functions:

```ocaml
let test_eval =
  (* we only care that [eval] doesn't raise an exception *)
  let _ = eval (`Int 5) in
  ignore (eval (`Add (`Int 1, `Int 2)))
```

There's a bug waiting to happen here. Suppose we later refactor `eval` to take another variable:

```ocaml
val eval : expr -> context -> expr
```

Suddenly, the test is ignoring partially-applied functions of type `context -> expr`: it silently became useless! For this reason, one often sees code that explicitly asserts the type of the value being ignored:

```ocaml
let ignore_expr (_ : expr) = ()

let test_eval =
  (* require the ignored value to have type [expr] *)
  let (_ : expr) = eval (`Int 5) in
  ignore_expr (eval (`Add (`Int 1, `Int 2)))
```

The type-checker now catches our partial application bug:

```text
File "test_eval.ml", line 9, characters 19-25:
9 |   let (_ : expr) = eval (`Int 5) in
                       ^^^^^^^^^^^^^
Error: This expression has type context -> expr
       but an expression was expected of type expr
```