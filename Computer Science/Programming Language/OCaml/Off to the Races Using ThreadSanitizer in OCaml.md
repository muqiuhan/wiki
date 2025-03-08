#ocaml #fp

OCaml Multicore opened up a new world of performance for developers, something that [Nomadic Labs has tested with great results.](https://tarides.com/blog/2022-12-20-how-nomadic-labs-used-multicore-processing-to-create-a-faster-blockchain/) Rather than relying on one core to do everything, the program can take advantage of multiple cores simultaneously for a significant performance boost.

With new programming possibilities come new classes of bugs, which require updated detection methods. One of these types of bugs is called a data race. A data race is a race condition that occurs when two accesses are made to the same memory location, at least one is a write, and no order is enforced between them.

Data races can be dangerous as they are easy to miss and capable of yielding unexpected results. Consequently, integrating a tool to detect data races has been a high priority for the teams working on OCaml 5.0 with Multicore support. Whilst data races in OCaml are less problematic than in many other languages (for example, data races in OCaml do not cause crashes and do not constitute undefined behaviour), developers still want to be made aware of possible data races so that they can remove them from their programs. More about this below.

## [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#what-is-tsan)

## What is TSan?

[ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html), or TSan, is an open-source tool that reliably detects data races at runtime. It consists of instrumenting programs with calls to a dedicated runtime that performs the detection.

Support for TSan will officially be part of the OCaml 5.2 release, and there is already a backport for OCaml 5.1.

This blog post will demonstrate the benefits of using TSan, offer insight into how TSan works, and outline the challenges of integrating it with OCaml. For a more practically oriented guide on how to use TSan in your own projects, the [tutorial on using TSan with OCaml Multicore](https://ocaml.org/docs/multicore-transition) is a great place to start.

We will begin by examining what a data race looks like, both before and after using TSan.

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#a-practical-example)

### A Practical Example

Let us consider how a data race might occur. Say an OCaml programmer writes code to populate a table of clients from several sources. They decide to make it Multicore to improve performance by using two `Domain`s for two data sources:

```ocaml
let clients = Hashtbl.create 16
let free_id = Atomic.make 0

let clients1 = (* Some data source *)

let clients2 = (* Some data source *)

let record_clients =
  Seq.iter
    (fun c -> Hashtbl.add clients (Atomic.fetch_and_add free_id 1) c)

let () =
  let d = Domain.spawn (fun () -> record_clients clients1) in
  record_clients clients2;
  Domain.join d
```

As we can tell, each incoming client is bound to a unique ID. The programmer correctly used the `Atomic` module for ID generation, ensuring the IDs are truly unique. However, they have failed to use a domain-safe module designed for concurrency, instead opting for `Hashtbl`. Unfortunately, this module is unsafe for concurrent use: using `Hashtbl.t` in parallel can cause data races and lead to surprising results.

For example, when two domains add elements in parallel it may cause some elements to be silently dropped. To make matters worse, the resulting bugs would be non-deterministic and as such be hard to detect and track down. Furthermore, if the programmer's project depends on libraries that use `Hashtbl`, it would make them unsafe to use in parallel without it necessarily being clear from their documentation.

If, however, the programmer were to build their program on a special `opam` switch with a TSan-enabled compiler like this:

```text
$ opam switch create 5.1.0+tsan
$ opam install dune
$ dune exec ./clients.exe
```

(Side note: the `5.1.0+tsan` switch is the most convenient way to use TSan with OCaml at the time of writing. Once OCaml 5.2 is released, the blessed command will be `opam switch create <switch name> ocaml-option-tsan`.)

All memory accesses would be instrumented with calls to the TSan runtime, and TSan would detect the data race condition and output a data race report:

```text
==================
WARNING: ThreadSanitizer: data race (pid=790576)
  Write of size 8 at 0x7f42b37f57e0 by main thread (mutexes: write M86):
    #0 caml_modify runtime/memory.c:166 (clients.exe+0x58b87d)
    #1 camlStdlib__Hashtbl.resize_749 stdlib/hashtbl.ml:152 (clients.exe+0x536766)
    #2 camlStdlib__Seq.iter_329 stdlib/seq.ml:76 (clients.exe+0x4c8a87)
    #3 camlDune__exe__Clients.entry /workspace_root/clients.ml:9 (clients.exe+0x4650ef)
    #4 caml_program <null> (clients.exe+0x45fefe)
    #5 caml_start_program <null> (clients.exe+0x5a0ae7)

  Previous read of size 8 at 0x7f42b37f57e0 by thread T1 (mutexes: write M90):
    #0 camlStdlib__Hashtbl.key_index_1308 stdlib/hashtbl.ml:507 (clients.exe+0x53a625)
    #1 camlStdlib__Hashtbl.add_1312 stdlib/hashtbl.ml:511 (clients.exe+0x53a6f8)
    #2 camlStdlib__Seq.iter_329 stdlib/seq.ml:76 (clients.exe+0x4c8a87)
    #3 camlStdlib__Domain.body_703 stdlib/domain.ml:202 (clients.exe+0x50bf60)
    #4 caml_start_program <null> (clients.exe+0x5a0ae7)
    #5 caml_callback_exn runtime/callback.c:197 (clients.exe+0x56917b)
    #6 caml_callback runtime/callback.c:293 (clients.exe+0x569cb0)
    #7 domain_thread_func runtime/domain.c:1100 (clients.exe+0x56d37f)
    [...]

SUMMARY: ThreadSanitizer: data race runtime/memory.c:166 in caml_modify
==================
[...]
ThreadSanitizer: reported 2 warnings
```

Above is a truncated view of what the TSan report, warning of a data race, looks like in this case. TSan has detected two memory accesses, a write and a read, made to one memory location. As they are also unordered, this constitutes a data race and TSan reports it along with the backtraces of both accesses.

In this case it would be evident that something has gone wrong with `Hashtbl.add` – a big hint to the programmer.

## [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#under-the-hood)

## Under the Hood

Now that we know what TSan is used for, it's time to explore how it works. Compiling a program with TSan enabled causes the executable to be instrumented with calls to the TSan runtime library. The runtime library tracks memory accesses and ordering relations between these accesses.

Internally, the TSan runtime assigns a vector clock to each OCaml domain or system thread. Each thread holds a vector clock – a vector clock being an array of _n_ integers, where _n_ is the number of threads – and increments its clock upon each event (memory access, mutex operation, etc.). Certain operations like mutex locks, atomic reads, and so on, will synchronise clocks between threads.

[![A mutex lock synchronising the clock between two threads.](https://tarides.com/static/0bf64db362ac972b1285ab93f569b53a/c5bb3/vector-clocks.png)](https://tarides.com/static/0bf64db362ac972b1285ab93f569b53a/798d4/vector-clocks.png)

Comparing vector clocks allows TSan to establish an order between events, so-called [happens-before relations.](https://jameshfisher.com/2017/02/10/happened-before/) TSan reports a data race every time two memory accesses are made to overlapping memory regions, **if:**

- At least one of them is a write, and
- There is no established happens-before relation between them.

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#shadow-state)

### Shadow State

Let us look at this process in more detail. Each word of application memory is associated with one or more 'shadow words'. Each shadow word contains information about a recent memory access to that word. This information points to the vector clock's state at the moment the access was performed.

[![A box labelled application with an arrow to a box labeled shadow state.](https://tarides.com/static/805211c514eae2a5f8a8410fd26f476a/c5bb3/shadow-state.png)](https://tarides.com/static/805211c514eae2a5f8a8410fd26f476a/d125e/shadow-state.png)

This information (called the 'shadow state') is updated at every instrumented memory access: TSan compares the accessor's clock with each existing shadow word, and checks the following:

- Do the accesses overlap?
- Is one of them a write?
- Are the thread IDs different?
- Are they unordered by happens-before?

If these conditions are met, TSan detects and reports a data race.

In addition to memory access, operations like `Domain.spawn` and `Domain.join` (as well as mutex operations) are relevant for operation ordering. As such, TSan also instruments these operations.

## [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#integrating-tsan-with-ocaml)

## Integrating TSan with OCaml

The core of TSan support is instrumentation of memory acceses with calls to the TSan runtime. The OCaml compiler performs this instrumentation in a dedicated pass.

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#exceptions)

### Exceptions

For TSan to show a backtrace of past events, function entries and exits must also be instrumented. This is done as part of the instrumentation pass.

However, in OCaml, a function can also be exited by an [exception](https://github.com/fabbing/obts_exn), bypassing part of the instrumentation. When that happens, for TSan’s view of the backtrace to remain up-to-date, the OCaml runtime informs TSan about every exited function.

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#effect-handlers)

### Effect Handlers

[Effect handlers](https://v2.ocaml.org/manual/effects.html) are a generalisation of exception handlers. Performing an effect results in a jump to the associated effect handler, and then a delimited continuation makes it possible to resume the computation. In the same way as with exceptions, the OCaml runtime must signal to TSan which functions are exited when an effect is performed and re-entered when a continuation is resumed.

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#memory-model)

### Memory Model

Each language specifies how memory behaves in parallel programs using what is known as a memory model. Incidentally, what counts as a data race in a given language also depends on its memory model.

TSan can detect data races in programs that follow the C memory model. OCaml 5's memory model is different from the C model, however, and it offers more guarantees: data races in C and C++ cause _undefined behaviour_ (i.e., anything can happen), which is not the case in OCaml. OCaml's semantics are “fully defined” (see the [manual page](https://v2.ocaml.org/manual/memorymodel.html) about the memory model). In particular, a program with data races in OCaml will not crash, unlike in C++. In addition, there can be no [out-of-thin-air values](https://www.hboehm.info/c++mm/thin_air.html): the only values that can be observed are values that are previously written to that location. The OCaml memory model guarantees that even for programs with data races, memory safety is preserved.

Data races in OCaml can still result in unexpected surprises for the OCaml programmer. A multi-threaded execution may produce behaviours that cannot be explained by the mere interleaving of actions from different threads. The only way such behaviours can be explained is through a reordering of actions in the same thread. Such reasoning is quite unintuitive for a programmer who will be more used to thinking about program behaviour as being an interleaving of actions from different threads.

However, if the program is data-race free, then the observed behaviour can be explained by a simple interleaving of operations from different threads (a property known as _sequential consistency_). Eliminating data races reduces non-determinism in the program and hence it is beneficial to remove data races whenever possible. Note that we do not completely eliminate non-determinism from a parallel program.

In essence, because of the differences between the C and OCaml memory models, in order for TSan to detect data races in OCaml the instrumentation of memory accesses must _conceptually_ map OCaml programs to C programs. During development, the team took care to ensure that this mapping preserved the detection of data races (in the OCaml sense) and did not introduce false positives.

You can find more details about the inner workings of TSan and its OCaml support in this [OCaml Workshop 2023 talk](https://github.com/fabbing/ocaml_tsan_icfp/blob/master/presentation/presentation.pdf).

### [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#performance-and-limitations)

### Performance and Limitations

In terms of the cost of running TSan, currently, it affects memory and performance in the following ways:

- Performance cost: about a 2-7x slowdown (compared to 5-15x for C/C++)
- Memory consumption: increased by about 4-7x (compared to 5-10x for C++)

As with all tools, TSan has some limitations. These are due to how TSan is built and are unlikely to change. With TSan, data races are only detected on visited code paths. In addition, TSan only remembers a finite amount of memory accesses for space-saving reasons, which can in principle cause TSan to miss some races. TSan also does not currently support Windows.

TSan support for OCaml is currently only implemented for x86-64 Linux and macOS, but will hopefully be extended to include more architectures such as arm64.

## [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#use-cases)

## Use Cases

Knowing the limitations, let us explore TSan's use cases. So far, TSan has helped by unearthing data races in several OCaml libraries:

- [Saturn (formerly known as Lockfree):](https://github.com/ocaml-multicore/saturn) TSan [found a benign data race](https://github.com/ocaml-multicore/saturn/issues/39), as well as a data race occuring from the use of [semaphores](https://github.com/ocaml-multicore/saturn/pull/40).
- [Domainslib:](https://github.com/ocaml-multicore/domainslib) TSan found benign data races in Chan, not just [once](https://github.com/ocaml-multicore/domainslib/issues/72) but [twice](https://github.com/ocaml-multicore/domainslib/pull/103).
- [The OCaml runtime system](https://github.com/ocaml/ocaml) itself: TSan warned about a [number of race conditions](https://github.com/ocaml/ocaml/issues/11040) in the OCaml runtime.

In addition, TSan has been a great help in transitioning the effects-based I/O library [Eio](https://github.com/ocaml-multicore/eio) and the distributed database [Irmin](https://github.com/mirage/irmin) to Multicore. It allowed teams to detect potential data races and fix them as required.

## [](https://tarides.com/blog/2023-10-18-off-to-the-races-using-threadsanitizer-in-ocaml/#feedback)

## Feedback!

We want to hear from you – are you using TSan for your OCaml projects? Please get in touch and let us know about your experience, whether you have encountered any problems, and if you have any suggestions for how it could be improved.

You can share your thoughts on the [OCaml Discuss Forum](https://discuss.ocaml.org) or contact Tarides directly [on our website](https://tarides.com/contact/). Don't forget to check out the [TSan tutorial](https://ocaml.org/docs/multicore-transition) as well. Happy hacking!