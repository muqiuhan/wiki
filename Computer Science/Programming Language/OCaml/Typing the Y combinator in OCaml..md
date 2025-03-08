```ocaml
(* Typing the Y combinator in OCaml.
 * $ ocaml y.ml
   720
   Succ(Succ(Succ(Succ(Succ(Succ(Zero))))))
 *)

type 'a fix = Fix of ('a fix -> 'a)

let fix x = Fix x
let unfix (Fix x) = x

let y : 'a 'b. (('a -> 'b) -> 'a -> 'b) -> 'a -> 'b
  = fun f ->
  let g x a =
    f ((unfix x) x) a
  in
  g (fix g)

(* The factorial function. *)
let fact self n = if n = 0 then 1 else n * self (n - 1)

(* A representation of natural numbers. *)
type nat = Zero | Succ of nat

let int2nat self n = if n = 0 then Zero else Succ (self (n - 1))
let rec string_of_nat = function
  | Zero -> "Zero"
  | Succ n -> Printf.sprintf "Succ(%s)" (string_of_nat n)

let _ =
  let result = y fact 6 in
  Printf.printf "%d\n%!" result;
  let result = y int2nat 6 in
  Printf.printf "%s\n%!" (string_of_nat result)
```