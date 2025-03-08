#ocaml #fp

Currently OCaml 5 provides a [`Domain.DLS` 2](https://v2.ocaml.org/api/Domain.DLS.html) module for domain-local storage.

Unfortunately,

1. [there is no corresponding `Thread.TLS` 7](https://github.com/ocaml/ocaml/issues/11770) for (sys)thread-local storage, and
    
2. the current implementation of `Domain.DLS` is not thread-safe.
    

I don’t want to spend time to motivate this topic, but for many of the use cases of `Domain.DLS`, what you actually want, is to use a `Thread.TLS`. IOW, many of the uses of `Domain.DLS` are probably “wrong” and should actually use a `Thread.TLS`, because, when using `Domain.DLS`, the implicit assumption is often that you don’t have multiple threads on the domain, but that is typically decided at a higher level in the application and so making such an assumption is typically not safe.

## [](https://discuss.ocaml.org/t/a-hack-to-implement-efficient-tls-thread-local-storage/13264/1#domaindls-is-not-thread-safe-1)`Domain.DLS` is not thread-safe

I mentioned that the current implementation of `Domain.DLS` is not thread-safe. What I mean by that is that [the current implementation is literally not thread-safe at all](https://github.com/ocaml/ocaml/issues/12677) in the sense that unrelated concurrent `Domain.DLS` accesses can actually break the DLS. That is because the state updates performed by `Domain.DLS` contain safe-points during which the OCaml runtime may switch between (sys)threads.

Consider [the implementation of `Domain.DLS.get` 1](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/stdlib/domain.ml#L120-L127):

```ocaml
  let get (idx, init) =
    let st = maybe_grow idx in
    let v = st.(idx) in
    if v == unique_value then
      let v' = Obj.repr (init ()) in
      st.(idx) <- (Sys.opaque_identity v');
      Obj.magic v'
    else Obj.magic v
```

If there are two (or more) threads on a single domain that concurrently call `get` before `init` has been called initially, then what might happen is that `init` gets called twice (or more) and the threads get different values which could e.g. be pointers to two different mutable objects.

Consider [the implementation of `maybe_grow`](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/stdlib/domain.ml#L98-L111):

```ocaml
  let maybe_grow idx =
    let st = get_dls_state () in
    let sz = Array.length st in
    if idx < sz then st
    else begin
      let rec compute_new_size s =
        if idx < s then s else compute_new_size (2 * s)
      in
      let new_sz = compute_new_size sz in
      let new_st = Array.make new_sz unique_value in
      Array.blit st 0 new_st 0 sz;
      set_dls_state new_st;
      new_st
    end

```

Imagine calling `get` (which calls `maybe_grow`) with two different keys from two different threads concurrently. The end result might be that two different arrays are allocated and only one of them “wins”. What this means, for example, is that effects of `set` calls may effectively be undone by concurrent calls of `get`.

In other words, the `Domain.DLS`, as it is currently implemented, is not thread-safe.

## [](https://discuss.ocaml.org/t/a-hack-to-implement-efficient-tls-thread-local-storage/13264/1#how-to-implement-an-efficient-threadtls-2)How to implement an efficient `Thread.TLS`?

If you dig into the implementation of threads, you will notice that the opaque [`Thread.t` type](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/otherlibs/systhreads/thread.mli#L18) is actually a heap block (record) of three fields. You can see the [`Thread.t` accessors](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/otherlibs/systhreads/st_stubs.c#L66-L68):

```
#define Ident(v) Field(v, 0)
#define Start_closure(v) Field(v, 1)
#define Terminated(v) Field(v, 2)
```

and the [`Thread.t` allocation 1](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/otherlibs/systhreads/st_stubs.c#L335-L346):

```
static value caml_thread_new_descriptor(value clos)
{
  CAMLparam1(clos);
  CAMLlocal1(mu);
  value descr;
  /* Create and initialize the termination semaphore */
  mu = caml_threadstatus_new();
  /* Create a descriptor for the new thread */
  descr = caml_alloc_3(0, Val_long(atomic_fetch_add(&thread_next_id, +1)),
                       clos, mu);
  CAMLreturn(descr);
}
```

The second field, `Start_closure`, is used to pass the closure to the thread start:

```
static void * caml_thread_start(void * v)
{
  caml_thread_t th = (caml_thread_t) v;
  int dom_id = th->domain_id;
  value clos;
  void * signal_stack;

  caml_init_domain_self(dom_id);

  st_tls_set(caml_thread_key, th);

  thread_lock_acquire(dom_id);
  restore_runtime_state(th);
  signal_stack = caml_init_signal_stack();

  clos = Start_closure(Active_thread->descr);
  caml_modify(&(Start_closure(Active_thread->descr)), Val_unit);
  caml_callback_exn(clos, Val_unit);
  caml_thread_stop();
  caml_free_signal_stack(signal_stack);
  return 0;
}

```

and, as seen above, [it is overwritten with the unit value](https://github.com/ocaml/ocaml/blob/e397ed28bcef85fdc1f0f007af481ef201fb1fd7/otherlibs/systhreads/st_stubs.c#L575) before the closure is called.

What this means is that when you call `Thread.self ()` and get a reference to the current `Thread.t`, the `Start_closure` field of that heap block will be the unit value:

```ocaml
assert (Obj.field (Obj.repr (Thread.self ())) 1 = Obj.repr ())
```

Let’s hijack that field for the purpose of implementing an efficient TLS!

Here is the full hack:

```ocaml
module TLS : sig
  type 'a key
  val new_key : (unit -> 'a) -> 'a key
  val get : 'a key -> 'a
  val set : 'a key -> 'a -> unit
end = struct
  type 'a key = { index : int; compute : unit -> 'a }

  let counter = Atomic.make 0
  let unique () = Obj.repr counter

  let new_key compute =
    let index = Atomic.fetch_and_add counter 1 in
    { index; compute }

  type t = { _id : int; mutable tls : Obj.t }

  let ceil_pow_2_minus_1 n =
    let n = n lor (n lsr 1) in
    let n = n lor (n lsr 2) in
    let n = n lor (n lsr 4) in
    let n = n lor (n lsr 8) in
    let n = n lor (n lsr 16) in
    if Sys.int_size > 32 then n lor (n lsr 32) else n

  let[@inline never] grow_tls t before index =
    let new_length = ceil_pow_2_minus_1 (index + 1) in
    let after = Array.make new_length (unique ()) in
    Array.blit before 0 after 0 (Array.length before);
    t.tls <- Obj.repr after;
    after

  let[@inline] get_tls index =
    let t = Obj.magic (Thread.self ()) in
    let tls = t.tls in
    if Obj.is_int tls then grow_tls t [||] index
    else
      let tls = (Obj.magic tls : Obj.t array) in
      if index < Array.length tls then tls else grow_tls t tls index

  let get key =
    let tls = get_tls key.index in
    let value = Array.unsafe_get tls key.index in
    if value != unique () then Obj.magic value
    else
      let value = key.compute () in
      Array.unsafe_set tls key.index (Obj.repr (Sys.opaque_identity value));
      value

  let set key value =
    let tls = get_tls key.index in
    Array.unsafe_set tls key.index (Obj.repr (Sys.opaque_identity value))
end
```

The above achieves about 80% of the performance of `Domain.DLS` allowing roughly 241M `TLS.get`s/s (vs 296M `Domain.DLS.get`s/s) on my laptop.