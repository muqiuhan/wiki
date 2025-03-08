#fp 

We describe interesting ways of representing an inherently sequential accumulation (`fold`) as the composition of a `map` and a monoid reduction. As the result, some seemingly sequential algorithms can run not just in parallel but embarrassingly in parallel: The input sequence can be arbitrarily partitioned (and recursively sub-partitioned, if desired) among workers, which would run in parallel with no races, dependencies or even memory bank conflicts. Such embarrassing parallelism is ideal for multi-core, GPU or distributed processing.

The general principle is well-known; also well-known is that its specific applications require ingenuity.

- [Introduction: Fold v. Monoid Reduction](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#intro)
- [Fold as map-reduce, trivially](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#trivial)
- [Fold as map-reduce, more interestingly](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#simple)
- [Generalized Horner rule](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#Horner)
- [Boyer-Moore majority voting](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#BM)
- [Conclusions](https://okmij.org/ftp/Algorithms/map-monoid-reduce.html#conclusions)

---

## Introduction: Fold v. Monoid Reduction

At ICFP 2009, Guy Steele gave a keynote calling ``foldl and foldr considered slightly harmful'' and advocating (map)reduce instead.

Recall, folding over a sequence is an inherently sequential stateful accumulation; using lists for concreteness, it is defined as

    fold_left  : ('z -> 'a -> 'z) -> 'z -> 'a list -> 'z
    fold_right : ('a -> 'z -> 'z) -> 'a list -> 'z -> 'z

Actually, there are two operations: the left and the right fold. Their meaning should be clear from the following example:

    fold_left (+) 0 [1;2;3;4]  ≡ (((0 + 1) + 2) + 3) + 4
    
    fold_right (+) [1;2;3;4] 0 ≡ 1 + (2 + (3 + (4 + 0)))

In this case, of the folding function being addition, the results are identical: both expressions sum the list. Generally, left and right folds produce different results: try, for example, subtraction as the folding function. The types of the list elements `'a` and of the accumulator `'z` need not be the same: for example,

    fold_left (fun z _ -> z + 1) 0 l

computes the length of any list,

    fold_left (fun z x -> x :: z) [] l

reverses the list `l` and

    fold_right (fun x z -> if p x then x::z else z) l []

filters the list: omits the elements for which the predicate `p` returns `false`. Many other operations on lists (in fact, all of them) can be expressed as folds. Fold is indeed the general pattern of sequential stateful processing of a sequence.

The are alternative definitions of fold: sometimes the last two arguments of the right fold are swapped. If the arguments of the right folding function are likewise swapped, then the left and the right fold have the same signature. Their behavior, the association pattern, is still different. Olivier Danvy, see below, traces the history of list folds and the argument order.

For concreteness we showed folds over lists, but similar operations exist over arrays, streams, files, trees, dictionaries and any other collections.

In this article, by reduce we always mean the reduce over a monoid. A monoid is a set (called `carrier set') with an associative binary operation which has a unit (also called zero) element. Concretely, in OCaml

    type 'a monoid = {zero: 'a; op: 'a -> 'a -> 'a}

where `'a` is the type of monoid elements, `op` must be associative and

    op zero x = op x zero = x

must hold for every element `x` of the monoid. In Google MapReduce, the operation `op` is also taken to be commutative. We do not impose such requirement.

Reduce over a sequence (here, a list) is the operation

    reduce : 'a monoid -> 'a list -> 'a

with the behavior that can be illustrated as

    reduce monoid []  ≡ monoid.zero
    reduce monoid [x] ≡ x             (* for any x and monoid *)
    
    reduce {zero=0;op=(+)} [1;2;3;4] ≡ 1 + 2 + 3 + 4

One may say that reduce `wedges in' the monoid operation between the consecutive elements of the sequence. Since `op` is associative, the parentheses are not necessary. Therefore, unlike the left and the right fold, there is only one reduce.

We shall see that reduce is often pre-composed with map. The two operations may always be fused, into

    map_reduce : ('a -> 'z) -> 'z monoid -> 'a list -> 'z

Nevertheless, we will write map and reduce separately, for clarity -- assuming that actual implementations use the efficient, fused operation.

Fold has to be evaluated sequentially, because of the data dependency on the accumulator. In fact, the left fold is just another notation for a `for`-loop:

    type 'a array_slice = {arr:'a array; from:int; upto:int} 
    let fold_left_arr : ('z -> 'a -> 'z) -> 'z -> 'a array_slice -> 'z = 
      fun f z {arr;from;upto} ->
      let acc = ref z in
      for i=from to upto do
        acc := f !acc arr.(i)
      done;
      !acc

Here, for variety, we use an array slice as a sequence.

On the other hand, reduce has a variety of implementations. It may be performed sequentially:

    let seqreduce_arr (m: 'a monoid) (arrsl: 'a array_slice) : 'a =
      fold_left_arr m.op m.zero arrsl

Or it can be done in parallel:

    let rec parreduce_arr (m: 'a monoid) {arr;from;upto} : 'a =
      match upto+1-from with
      | 0 -> m.zero
      | 1 -> arr.(from)
      | 2 -> m.op arr.(from) arr.(from+1)
      | 3 -> m.op (m.op arr.(from) arr.(from+1)) arr.(from+2)
      | n -> let n' = n / 2 in
             (* Here, the two parreduce_arr invocations can be done in parallel! *)
             m.op 
               (parreduce_arr m {arr;from;upto=from+n'-1})
               (parreduce_arr m {arr;from=from+n';upto})

The two `parreduce_arr` invocations in the recursive case can run in parallel -- embarrassingly in parallel, with no races or even read dependencies. Whether they should be done in parallel is another question -- something that we can decide case-by-case. For example, if the array slice is short, the two `parreduce_arr` invocations are better done sequentially (since the ever-present overhead of parallel evaluation would dominate.) If the two halves of the slice are reduced also in parallel, we obtain a hierarchical decomposition: binary-tree--like processing.

We do not have to recursively decompose the slices. We may cut the input array into a sequence of non-overlapping slices, arbitrarily, and assign them to available cores. The cores may do their assigned work as they wish, without any synchronization with the others. At the end, we combine their results using the monoid operation.

Since the (left or right) fold commits us to sequential evaluation, Guy Steele called it `slightly harmful'. He urged using reduce, as far as possible, because it is flexible and decouples the algorithm from the execution strategy. The strategy (sequential, parallel, distributed) and data partitioning can be chosen later, depending on circumstances and available resources.

Thus the main question is: can we convert fold into reduce? The rest of the article answers it.

**References**

Guy Steele: Organizing Functional Code for Parallel execution or, foldl and foldr considered slightly harmful  
August 2009 (ICFP 2009, Keynote)

Olivier Danvy: ``Folding left and right matters: direct style, accumulators, and continuations''  
Journal of Functional Programming, vol 33, e2. Functional Pearl, February 2023  
Appendix A: A brief history of folding left and right over lists

According to Danvy, the first instances of `fold_left` and `fold_right` were investigated by Christopher Strachey in 1961(!). Reduce is part of APL (Iverson, 1962), where it is called `/': hence `+/x` sums the array `x`. Goedel recursor R in System T is a more general version of fold (called now para-fold). Church numerals are folds.

[monoid_reduce.ml](https://okmij.org/ftp/Algorithms/monoid_reduce.ml) [12K]  
Complete code for the article

[How to zip folds](https://okmij.org/ftp/Streams.html#zip-folds)  
A complete library of fold-represented lists, demonstrating that all list processing operations can be expressed as folds

[Accumulating tree traversals, a better tree fold](https://okmij.org/ftp/Scheme/xml.html#Papers) with applications to XML parsing

## Fold as map-reduce, trivially

The main question is expressing `fold_left` and `fold_right` in terms of `reduce`. In this section we see two trivial answers. In fact, we see that fold can _always_ be expressed in terms of reduce -- but in a way that is not useful or interesting.

If the folding function, the first argument of fold, is associative (which implies that the type of sequence elements is the same as the accumulator/result type) and has a zero element, then fold is trivially an instance of reduce. We have seen the example already:

    fold_left (+) 0 l ≡ fold_right (+) l 0 ≡ reduce {op=(+);zero=0} l

for any list `l`.

The second trivial answer is that a fold can always be expressed in terms of a monoid reduce, unconditionally:

    let compose_monoid = {op=(fun g h -> fun x -> g (h x)); zero=Fun.id}
    fold_right f l z ≡ List.map f l |> reduce compose_monoid |> (|>) z

for any `f`, `z`, `l` of the appropriate types. The very similar expression can be written for the left fold (left as an exercise to the reader). The key idea is that function composition is associative.

Unfortunately, this trivial answer is of little practical use. If `f:'a -> 'z -> 'z`, then `map f l` creates a list of `'z -> 'z` closures, which `reduce` composes together into a (big) `'z -> 'z` closure, which is finally applied to `z`. If the list `l` is of size `N`, the result of the reduction is the composition of `N` closures, and hence has the size proportional to `N` (with a fairly noticeable proportionality constant: a closure takes several words of heap space). In effect we have built an intermediate data structure of unbounded size. Although composing the closures can be parallelized, the useful work is done only when the big closure composition is finally applied to the initial accumulator `z` -- at which point the folding function `f` is applied step-by-step, sequentially, just like in the original `fold_right f l z`. This trivial reduction of fold only wastes time and space.

The problem hence is not merely to express fold as reduce, but do so efficiently, without undue and unbounded overhead. Only then we can profitably fold in parallel.

## Fold as map-reduce, more interestingly

Thus our problem is to _efficiently_ express fold as reduce, without intermediary data of unbound size and without taking undue time. In other words, if folding can be done in constant time per sequence element and in constant (working) space, so should reduce.

Sometimes the problem can be solved simply. We have already seen one such case: the folding function is associative and has the unit element. The left and the right fold are instances of reduce then. A little more interesting is the case of the folding function f that can be factored out in terms of two other functions `op` and `g` as

    f z x = op z (g x)

for any sequence element `x` and accumulator `z`. Here `op` is an associative operation that has a zero element. As an example, remember finding the length of a list, expressed as fold as

    List.fold_left (fun z _ -> z + 1) 0 l

The folding function indeed factors out

    z + 1 = z + (Fun.const 1 x)

and so length can be re-written as

    map (Fun.const 1) l |> reduce {op=(+);zero=0}

or, as `map_reduce`. Length, therefore, may be computed in parallel.

Such a factorization is an instance of the general principle, called ``Conjugate Transform'' in Guy Steele's talk. The principle calls for representing fold as

    fold_left (f:'z->'a->'a) (z:'z) l = map g l |> reduce m |> h

and similarly for the right fold. Here `m:'u monoid` is a monoid for some type `'u`, and `g:'a->'u` and `h:'u->'z` are some functions. They generally depend on `f` and `z`. Informally, the principle recommends we look for a ``bigger'' type `'u` that can embed both `'a` and `'z` and admits a suitable associative operation with a unit element.

One may always represent fold in such a way, as we saw in the previous section: we chose `compose_monoid` as `m`, with `'z->'z` as the type `'u`.

The conjugate transform is a principle, or a schema. It does not tell how to actually find the _efficient_ monoid, or if it even exists. In fact, finding the efficient monoid is often non-trivial and requires ingenuity. As an example, consider a subtractive folding, mentioned in passing earlier. Subtraction is not associative and therefore `fold_left (-)` is not an instance of reduce. A moment of thought however shows that

    fold_left (-) z l = z - fold_left (+) 0 l = z - reduce {op=(+);zero=0} l

In terms of the conjugate transform schema, the function `g` is negation and `h` is adding `z`. The second moment of thought shows that `fold_right (-)` can not be reduced just as easily (or at all?). Actually, the right-subtractive-folding can also be carried out efficiently in parallel, as a monoid reduction. The reader is encouraged to find this monoid -- to appreciate the difficulty and ingenuity.

## Generalized Horner rule

Horner rule, or schema, is a widely-used sequential algorithm to efficiently evaluate a polynomial. A similar accumulating pattern also frequently occurs in parsing. Although the algorithm is inherently sequential, on the face of it, it turns out representable as a monoid reduction. Our reduction is more general and flexible than the other known ways to parallelize the Horner rule.

As an illustrating example, we take the conversion of a sequence of digits (most-significant first) to the corresponding number -- a simple parsing operation. It is an accumulating sequential operation and can be written as fold:

    let digits_to_num = 
      List.fold_left (fun z x -> 10*z + (Char.code x - Char.code '0')) 0

For example, `digits_to_num ['1'; '7'; '5']` gives `175`. One can easily factor it as a composition of a `map` and

    List.fold_left (fun z x -> 10*z + x) 0

which is essentially the Horner rule: evaluating the polynomial Σ `d`i `b`i at `b=10`. The folding function `fun z x -> 10*z + x` is not associative, however.

Still, there is a way to build a monoid. It is the monoid with the operation

    let op (x,b1) (y,b2) = (x*b2+y, b1*b2)

which is clearly associative (but not commutative):

    ((x,b1) `op` (y,b2)) `op` (z,b3)
    = (x*b2+y, b1*b2) `op` (z,b3)
    = (x*b2*b3+y*b3+z, b1*b2*b3)
    = (x,b1) `op` (y*b3+z,b2*b3)
    = (x,b1) `op` ((y,b2) `op` (z,b3))

with the unit `(0,1)`:

    (x,b) `op` (0,1) = (0,1) `op` (x,b) = (x,b)

Therefore, `digits_to_num` can be written as

    let digits_to_num = 
        List.map (fun x -> (Char.code x - Char.code '0')) >>
        List.map (fun x -> (x,10)) >>
        reduce {op;zero=(0,1)} >> fst

where `>>` is the left-to-right functional composition. Our construction is also an instance of the conjugate transform, with `g` being `fun x -> (x,10)` and and `h` being `fst`.

Thus, the Horner rule can be expressed as map-reduce and can hence be evaluated embarrassingly in parallel.

Our construction is general and applies to any folding function `f : 'z -> 'a -> 'z` of the form

    f z x = m.op (h z) x

where `m.op` is associative and has a zero element (that is, an operation of some monoid `m`). This factorization looks very similar to the one in the previous section; the difference, however slight, makes finding the monoid quite more difficult. It does exists:

    let hmonoid (m:'a monoid) : ('a * ('a->'a)) monoid = 
      {op = (fun (x,h1) (y,h2) -> (m.op (h2 x) y, h1 >> h2));
       zero = (m.zero, Fun.id)}

Its carrier is the set of pairs `(x,h)` of monoid `m` elements and functions on them. We assume the composition `h1>>h2` is represented efficiently. For example, when `h` is a multiplication by a constant, it can be represented by that constant; the composition of such functions is then representable by the product of the corresponding constants.

The Horner rule has the feel of the parallel prefix problem and the famous parallel scan of Guy Blelloch (PPoPP 2009). Horner rule does not require reporting of intermediate results, however. Our monoid reduction differs from Blelloch's even-odd interleaving. It also differs from monoid-cached trees from Guy Steele's ICFP09 keynote. We do not rely on any special data structures: We can work with plain arrays, partitioning them across available cores.

Since Horner method is pervasive in polynomial evaluation, there are approaches to parallelize it. The most common is even/odd splitting of polynomial coefficients. Although it may be well suitable for SIMD processing, the even/odd splitting is bad for multicore since it creates read contention (bank conflicts) and wastes cache space. Our monoid reduce lets us assign different memory banks to different cores, for their exclusive, conflict-free access.

On the other hand, Estrin's scheme may be seen as a very particular instance of our monoid construction: binomial grouping. Our monoid reduction allows for arbitrary grouping, hierarchically if desired, and of not necessarily of the same size.

**References**

[monoid_reduce.ml](https://okmij.org/ftp/Algorithms/monoid_reduce.ml) [12K]  
Complete code for the article

## Boyer-Moore majority voting

Boyer-Moore majority voting is the algorithm for finding the majority element in a sequence -- that is, the element that occurs more than half of the time. The algorithm requires one pass over the sequence and the constant amount of working space. It is important to keep in mind that the return value is the majority element _if it exists_. The algorithm _in general_ cannot tell if the sequence has majority: if majority does not exist, the return value is an arbitrary element, not necessarily the most frequently occurring. In general, therefore, one needs the second pass over the sequence to count the occurrences of the returned element and verify it is indeed the majority. The second pass may be avoided or be unnecessary in some cases.

The Boyer-Moore algorithm is called a prototypical streaming algorithm and is inherently sequential. It may be surprising, therefore, that it may be presented as monoid reduction and hence performed just as efficiently in parallel -- in fact, embarrassingly in parallel. The monoid reduction implementation is actually simple. What is complex is convincing ourselves that it indeed works and does the right thing, in all cases.

The Boyer-Moore algorithm is a state machine processing, ingesting input element-by-element. It can hence be implemented as fold (assuming the input is given as a non-empty list):

    let bm : 'a list -> 'a = function h::t -> 
      let fsm (c,m) x =
          if c = 0 then (1,x)
          else if m = x then (c+1,m)
          else (c-1,m)
      in List.fold_left fsm (1,h) t >> snd

The state is the counter `c` and the candidate majority `m`. At the end of the algorithm, when the entire sequence is scanned, `m` is the majority element (if the sequence has one). If the counter `c` at the end is more than half of the sequence length, the sequence has majority and `m` is the majority element. Otherwise, we have to make another pass through the sequence to verify that `m` is indeed the majority. The verification pass may be unnecessary if we have a prior knowledge that majority exists -- or if we are satisfied with any element if it does not.

Although the algorithm seems inherently sequential, it can in fact be represented as monoid reduction. The monoid is simple and efficient:

    let bm_monoid (z:'a) : (int * 'a) monoid = 
      {zero = (0,z);
       op = fun (c1,m1) (c2,m2) ->
         if c1 = 0 then (c2,m2) else
         if c2 = 0 then (c1,m1) else
         if m1 = m2 then (c1+c2,m1) else
         if c1 > c2 then (c1-c2, m1) else
         (c2-c1, m2)}

Monoid elements (the carrier set) are pairs: of a natural number `c` and of a sequence element `m`. When `c` is zero, `m` may be arbitrary; in the above code we use `z` for such an arbitrary element. We could have avoided specifying it by defining a more sophisticated data type for monoid carrier, or using `'a option`. The reasoning below would get messier; therefore, we keep the above definition for the sake of clarity.

One can see that `op` is commutative and that `zero` is indeed the zero of `op`. However, is `op` associative? That is, is `bm_monoid` really a monoid? It is easy to check:

    let m1 = (2,10) and m2 = (3,9) and m3 = (2,8)
    op (op m1 m2) m3  ⇝ (1, 8) 
    op m1 (op m2 m3)  ⇝ (1, 10)

Unfortunately, `bm_monoid` is not a monoid.

To cut the suspense, it turns out that `bm_monoid` is `morally' a monoid, and reduction with it indeed gives the right result, in all cases. Proving it however is not that straightforward.

For the sake of proof, we extend `bm_monoid` to:

    let bm_monoid_ext (z:'a) : (int * 'a * ('a*'a) list) monoid = 
      {zero = (0,z,[]);
       op = fun (c1,m1,l1) (c2,m2,l2) ->
         if c1 = 0 then (c2,m2,l1@l2) else
         if c2 = 0 then (c1,m1,l1@l2) else
         if m1 = m2 then (c1+c2,m1,l1@l2) else
         if c1 > c2 then (c1-c2, m1, repeat c2 (m1,m2) @ l1 @ l2) else
         (c2-c1, m2, repeat c1 (m1,m2) @ l1 @ l2)}

Here, the infix operation `@` is list concatenation and `repeat (n:int) (x:'a) : 'a list` produces the list with of length `n` with elements all equal to `x`.

The carrier set of `bm_monoid_ext` is a set of triples: a natural number `c`, a sequence element `m`, and a list of pairs with distinct components (that is, `fst` of any pair is not equal to its `snd`). We note that `op` preserves the invariant that all pairs in `l` have distinct components.

Let us introduce the relation ≈ on that set, defined as follows: `x ≈ y` just in case `flatten x` is equal to `flatten y` modulo permutation, where

    let flatten : (int * 'a * ('a*'a) list) -> 'a list = fun (c,m,l) ->
      repeat c m @ List.map fst l @ List.map snd l

That is, `flatten x` collects all elements mentioned within `x` in a _multiset_. Therefore, `x ≈ y` iff the multiset of elements of `x` is equal to the multiset of elements of `y`. One easily sees that the relation `≈` is reflexive, commutative and associative. That is, it is an equivalence relation. It also has useful for us properties, as follows.

Proposition: `bm_monoid_ext` is a monoid modulo `≈`. That is,

    op (0,z,[]) (c,m,l) ≈ op (c,m,l) (0,z,[]) ≈ (c,m,l)
    op (c1,m1,l1) (op (c2,m2,l2) (c3,m3,l3)) ≈ op (op (c1,m1,l1) (c2,m2,l2)) (c3,m3,l3)

The associativity property follows from the fact `op` preserves elements occurring in `(c,m,l)`, including their multiplicities (and the multiset equality is associative).

Moreover, `≈` is _adequate_ for the task of finding the majority.

Proposition (Adequacy): if `(c1,m1,l1) ≈ (c2,m2,l2)` and the multiset `flatten (c1,m1,l1)` has majority, then `c1>0`, `c2>0`, and `m1=m2`. Informally, `≈` preserves the found majority (if the majority exists). The proposition is the consequence of the fact that majority is invariant to sequence permutation, and the following representation lemma.

Lemma (Representation): if a sequence (multiset) `flatten (c,m,l)` has majority, then `c>0` and `m` is the majority element.

For the proof we note that if the list of distinct pairs `l` has `n` elements, it has `2*n` of values. Not all of them are distinct. However, there is no value that occurs more than `n` times; otherwise, by the pigeonhole principle, some pair in `l` would have had equal components. Therefore, no value occurring in `(c,m,l)` that is distinct from `m` (if `c>0`) can be the majority element. If the majority exists, it has to be `m`, and `c>0`.

Our proofs are inspired by Boyer and Moore's original proofs. They however used induction (which is appropriate for the sequential algorithm). We, in contrast, used equational reasoning (and the pigeonhole principle) -- but no induction.

As an aside, it may seem that induction is the only proof method used in computer science -- many widely used textbooks present nothing but induction. In my opinion, induction is overused. Mathematics has many other proof methods.

The final step is to observe that the part of `bm_monoid_ext` that we actually care about -- the element `m` and the count `c` -- is computed with no regard to the list `l`. The list is a `ghost' list, so to speak, needed only for proving correctness but not for computing the majority. Therefore, we may omit it and recover the `bm_monoid`.

In the upshot, the majority voting may hence be performed as map-reduce: mapping with `(fun x -> (1,x))` and then reducing over the monoid `bm_monoid`. It is just as efficient as the original Boyer-Moore algorithm: one pass over the sequence using constant and small working space. However, the input may be split arbitrarily among available workers, who would do the search in parallel, without any interference or dependencies. Majority voting can be done embarrassingly in parallel.

**References**

<[https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_majority_vote_algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_majority_vote_algorithm)>

[monoid_reduce.ml](https://okmij.org/ftp/Algorithms/monoid_reduce.ml) [12K]  
Complete code for the article

## Conclusions

In his ICFP'09 keynote, Guy Steele exhorted us to use reduce (or, `map_reduce`) rather than fold, as far as possible. Unlike fold, reduce does not commit us to a particular evaluation strategy. It can be performed sequentially, embarrassingly parallel, or in a tree-like fashion. Like the Σ notation in Math, it specifies what should be summed up, but not how or in which sequence.

Guy Steele conclusions apply to the present article as well. Especially his final thoughts:

- Associative combining operators are a VERY BIG DEAL
- Inventing (and proving) new combining operators is a very, very big deal

---

### Last updated August 1, 2024

This site's top page is [**http://okmij.org/ftp/**](http://okmij.org/ftp/)  

oleg-at-okmij.org  
Your comments, problem reports, questions are very welcome!

Generated by MarXere