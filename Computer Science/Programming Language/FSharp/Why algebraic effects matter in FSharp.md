#fsharp #fp 

# Side effects

Side effects are impossible to avoid in imperative code, but they can make reasoning about the behavior of a program very difficult. F# allows us to use imperative side effects, but it's often better not to. How can we avoid side effects while still implementing effectful requirements?

As an example, let's take a very simple and common effectful example: logging. Say we're writing a function that computes the length of the hypotenuse of a right triangle from the lengths of the other two sides:  

```fsharp
let hypotenuse a b =
    printfn "Side a: %g" a
    printfn "Side b: %g" b
    let c = sqrt <| (a*a + b*b)
    printfn "Side c: %g" c
    c
```

Every time this function is called, we log the input and output to the console as a side effect. This is fine in a single-threaded application, but what happens if `hypotenuse` is called simultaneously from two different threads? That's trouble.

A common approach in the imperative/OO world is to use dependency injection (or plain old interfaces) to separate the logging API from its implementation. However, the resulting code still causes side effects. Is it possible to define an effectful `hypotenuse` function that doesn't have any side effects? It sounds almost like a contradiction in terms, but it can be done, and the solution is very interesting.

# Algebraic effects

The approach is to explicitly declare effects using an `Effect` type:  

```fsharp
type Effect<'result> =
    | Log of string * (unit -> Effect<'result>)
    | Result of 'result
```

In this simple example, there are only two effects:

- `Log` is what we use to write a string to a logger. This constructor takes an additional continuation function that is to be executed after the string is logged.
- `Result` simply holds a value and doesn't cause an effect.

Note that this type is an example of the [free monad](https://dev.to/shimmer/monads-for-free-in-f-30dl). `Log` corresponds to the `Free` constructor and `Result` corresponds to `Pure`. The free monad is useful here because it can chain effects together. For example, we could rewrite our calculation like this:  

```fsharp
let hypotenuse a b =
    Log ((sprintf "Side a: %g" a), fun () ->
        Log ((sprintf "Side b: %g" b), fun () ->
            let c = sqrt <| (a*a + b*b)
            Log ((sprintf "Side c: %g" c), fun () ->
                Result c)))
```

It's important to understand that this version of the function returns an `Effect<float>` rather than a `float` itself. You can think of this type as an "effectful" computation that will return a `float` when it is executed. However, until it is executed it does nothing - in particular, it has no side effects. It simply defines a computation.

# Handling effects

In order to actually compute a result, we need some additional code that can "handle" our effects, just like an exception handler handles exceptions (which are also a kind of effect). Let's write a handler that accumulates log messages in a list while performing a calculation:  

```fsharp
let handle effect =

    let rec loop log = function
        | Log (str, cont) ->
            let log' = str :: log
            loop log' (cont ())
        | Result result -> result, log

    let result, log = loop [] effect
    result, log |> List.rev
```

When we pass an effectful computation to `handle`, we get two things back: the final result of the computation, and a list of all the log messages that were written during the computation:  

```fsharp
let c, log =
    hypotenuse a b
        |> handle
```

Note that `handle` is also a pure function - it doesn't write anything to the console or perform any other side effect. If we want, we could then write the resulting log to the console, but we'd have to be careful at that point to consider the actual side effects involved. The important thing is that we've successfully separated a pure functional calculation from the impure side effects of writing a log to the console. By solving those two problems separately, we've made it much easier to understand how our program behaves.

# Syntactic sugar

Of course, no one wants to write ugly nested `Log` invocations like this because they completely distract from the logic of the computation itself. Fortunately, we know that the free monad can help us here by providing a workflow:  

```fsharp
let rec bind f = function
    | Log (str, cont) ->
        Log (str, fun () ->
            cont () |> bind f)
    | Result result -> f result

type EffectBuilder() =
    member __.Return(value) = Result value
    member __.Bind(effect, f) = bind f effect

let effect = EffectBuilder()
```

This is the standard `bind` implementation for the free monad: it simply passes the binding function down the chain until the end, at which time the two effects are bound together by applying the function.

We also need some helper functions that "lift" a log string into the monad:  

```fsharp
let log str = Log (str, fun () -> Result ())
let logf fmt = Printf.ksprintf log fmt
```

Again, this follows the same pattern we've seen before with the free monad. With these tools in hand, we can rewrite our computation much more elegantly:  

```fsharp
let hypotenuse a b =
    effect {
        do! logf "Side a: %g" a
        do! logf "Side b: %g" b
        let c = sqrt <| (a*a + b*b)
        do! logf "Side c: %g" c
        return c
    }
```

This version of the function produces an `Effect<float>` that is identical to the previous one. It's just much easier to understand, and is essentially no more complex than the original version of `hypotenuse` that wrote directly to the console.

# Limitations

We can easily support additional effects by adding union cases to our `Effect` type. However, this sort of master list of all effects in a system isn't very practical. Ideally, we'd like to modularize effects so that they can be composed together. For example, we'd like to be able to handle log effects separately from exception effects and separately from stateful effects. Unfortunately, this isn't particularly easy to do in F# yet, but there is a library called [`Eff`](https://github.com/palladin/Eff) that serves as a proof of concept. (I wouldn't use it in production, though, because the handlers are rather ugly.)

In the future, I expect that algebraic effects will become mainstream, and support for explicit effect types will be baked into both functional and imperative languages. That's still several years away, though, but at least now you know it's (probably) coming.