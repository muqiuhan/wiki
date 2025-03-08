#fsharp #fp #software-engineering 

## Introduction 介绍

The "Model View Update" (MVU) architecture was made famous by the front-end programming language [Elm](http://elm-lang.org/) and can be found in many popular environments like [Redux](http://redux.js.org/). Today it's probably the most famous UI pattern in functional programming. The reason for this is that it clearly separates state changes from UI controls and provides a nice and easily testable way to create modular UIs. Especially for developers that are already familiar with concepts like CQRS or event sourcing many of the things in the MVU architecture will feel natural. Here's what the MVU pattern looks like:  
“模型视图更新”（MVU）架构因前端编程语言 Elm 而闻名，并且可以在 Redux 等许多流行环境中找到。如今，它可能是函数式编程中最著名的 UI 模式。原因是它清楚地将状态更改与 UI 控件分开，并提供了一种很好且易于测试的方法来创建模块化 UI。特别是对于已经熟悉 CQRS 或事件源等概念的开发人员来说，MVU 架构中的许多内容都会感觉很自然。 MVU 模式如下所示：

![](https://www.compositional-it.com/wp-content/uploads/2019/09/elmish-1.png)

**Elmish for F#** is a project that provides the capability for client-side apps written in F# to follow the MVU architecture. It comes in multiple flavors, so that you can choose between a [React renderer](https://fable-elmish.github.io/react/) for HTML, [React Native renderer](https://fable-elmish.github.io/react/react-native.html) for mobile apps and a [WPF renderer](https://github.com/Prolucid/Elmish.WPF) for desktop applications. In this article we will explore its power and modularity by working through a simple example.  
Elmish for F# 是一个为用 F# 编写的客户端应用程序提供遵循 MVU 架构的功能的项目。它有多种风格，因此您可以在用于 HTML 的 React 渲染器、用于移动应用程序的 React Native 渲染器和用于桌面应用程序的 WPF 渲染器之间进行选择。在本文中，我们将通过一个简单的示例来探索它的强大功能和模块化性。

## A simple counter 一个简单的计数器

Let's start to illustrate Elmish and the MVU architecture with the very common HTML button counter sample. The following code shows a counter implemented in HTML and JavaScript.  
让我们开始用非常常见的 HTML 按钮计数器示例来说明 Elmish 和 MVU 架构。以下代码显示了用 HTML 和 JavaScript 实现的计数器。

```javascript
<html>
    <body>
    <button onclick="--counter; update();">-</button>
    <div id="counter"></div>
    <button onclick="++counter; update();">+</button>
    <script>
        var counter = 0;

        function update() { 
        document.getElementById("counter").textContent = "" + counter;
        }

        update();
    </script>
    </body>
</html>
```

The code is very straight forward and works as intended, but it has a number of issues:  
该代码非常简单并且按预期工作，但它有许多问题：

1. We are mutating the global variable `counter` - almost always a dangerous thing to do  
    我们正在改变全局变量 `counter` - 几乎总是一件危险的事情
2. We are directly mutating a DOM element, coupling our "business logic" to the UI.  
    我们直接改变 DOM 元素，将我们的“业务逻辑”耦合到 UI。
3. We are referencing the DOM element with its name via a string, which is fragile and can lead to costly knock-on effects.  
    我们通过字符串引用 DOM 元素及其名称，这是脆弱的，可能会导致代价高昂的连锁反应。
4. We've embedded some "domain logic" directly in the `onclick` event  
    我们直接在 `onclick` 事件中嵌入了一些“领域逻辑”

These issues prevent us from using this in a modular way. For example, if we wanted to create a _list_ with counters, we could not reuse this code. Instead, we could copy & paste the code a couple of times to create a fixed number of counters, but even then we would need to be very careful that we fix all the references to the corresponding global variables. In the object-oriented world, there are number of patterns that allow you encapsulate this problem of shared mutable state - let's see how to do it in a manner that promotes some FP core practices using the three parts of the MVU pattern:  
这些问题阻止我们以模块化的方式使用它。例如，如果我们想创建一个带有计数器的列表，我们就不能重用这段代码。相反，我们可以复制并粘贴代码几次来创建固定数量的计数器，但即使如此，我们也需要非常小心地修复对相应全局变量的所有引用。在面向对象的世界中，有许多模式允许您封装共享可变状态的问题 - 让我们看看如何使用 MVU 模式的三个部分来促进一些 FP 核心实践：

### Model 模型

Let's start with the **Model**.  
让我们从模型开始。

```fsharp
type Model = int

type Msg =
| Increment
| Decrement

let init() : Model = 0
```

In this F# code we capture the current value of the counter in a _domain type_ , before creating a _message type_ which can signal that we want to increment or decrement the counter. We also implement an `init` function that allows us to create the initial model for our application.  
在此 F# 代码中，我们在创建消息类型之前捕获域类型中计数器的当前值，该消息类型可以表明我们想要递增或递减计数器。我们还实现了一个 `init` 函数，它允许我们为应用程序创建初始模型。

### View 看法

The **View** part deals with the question of displaying controls on the screen. This is where we need to decide on a UI framework; in our case we've decided to stick with HTML, and so we will use the React renderer.  
View 部分处理在屏幕上显示控件的问题。这是我们需要决定 UI 框架的地方；在我们的例子中，我们决定坚持使用 HTML，因此我们将使用 React 渲染器。

```fsharp
let view model dispatch =
    div []
        [ button [ OnClick (fun _ -> dispatch Decrement) ] [ "-" ]
          div [] [ model.ToString() ]
          button [ OnClick (fun _ -> dispatch Increment) ] [ "+" ] ]
```

This is _valid F# code_ that uses the excellent [Fable.React bindings](https://github.com/fable-compiler/fable-react) to convert from F# into React JS. The syntax is still similar to our HTML version from the beginning, but instead of `<` and `>`, we are using `[` and `]`. This syntax is easy to learn and IDE tools like [Ionide](http://ionide.io/) provide code completion for it, so it's a natural fit for existing F# developers. An important observation is that we are no longer mutating state from within the `OnClick` handlers; instead we simply _dispatch_ one of the earlier defined messages into the system. This decouples our model from the view.  
这是有效的 F# 代码，它使用出色的 Fable.React 绑定从 F# 转换为 React JS。语法从一开始仍然与我们的 HTML 版本类似，但我们使用 `[` 和 `]` 代替 `<` 和 `>` 。 。这种语法很容易学习，并且 Ionide 等 IDE 工具为其提供了代码补全，因此它非常适合现有的 F# 开发人员。一个重要的观察结果是，我们不再从 `OnClick` 处理程序内部改变状态；相反，我们只是将先前定义的消息之一发送到系统中。这将我们的模型与视图分离。

### Update 更新

In the **Update** part, we define a state machine that represents our domain logic (or at least hooks into it).  
在更新部分，我们定义一个状态机来表示我们的域逻辑（或至少挂钩它）。

```fsharp
let update (msg:Msg) (model:Model) : Model =
    match msg with
    | Increment -> model + 1
    | Decrement -> model - 1
```

Here, we have defined an `update` function that will be called by Elmish _whenever a message is received_. In this very basic scenario there are only two cases to handle, and we can't really see the power of F#'s pattern matching yet. The most interesting observation is that we don't rely on _any mutable state_. Instead the update function takes a message and the _current_ model, and returns a completely _new version_ of the model. Since all the data is simply inside the model, this is very well testable - we can easily write a set of unit tests around the update function.  
在这里，我们定义了一个 `update` 函数，每当收到消息时 Elmish 都会调用该函数。在这个非常基本的场景中，只有两种情况需要处理，而且我们还不能真正看到 F# 模式匹配的强大功能。最有趣的观察是我们不依赖任何可变状态。相反，更新函数接受消息和当前模型，并返回模型的全新版本。由于所有数据都位于模型内部，因此非常容易测试 - 我们可以轻松地围绕更新函数编写一组单元测试。

### The glue 胶水

So far, we've not used any functionality from Elmish at all. The **Model** and **Update** parts are pure F# code, whilst the **View** part uses the [Fable.React](https://github.com/fable-compiler/fable-react) package. At this point all three parts are completely independent from each other - now Elmish will "wire" these up into an application:  
到目前为止，我们还没有使用 Elmish 的任何功能。模型和更新部分是纯 F# 代码，而视图部分使用 Fable.React 包。此时，所有三个部分完全相互独立 - 现在 Elmish 将把它们“连接”到一个应用程序中：

```fsharp
Program.mkSimple Counter.init Counter.update Counter.view
|> Program.withConsoleTrace
|> Program.withDebugger
|> Program.withReact "counter-app"
|> Program.run
```

Elmish's [Program](https://fable-elmish.github.io/elmish/program.html) abstraction provides "glue" functions like `mkSimple` to bind the three parts together into an application. Internally, Elmish gives us a message loop and takes care of dispatching messages between the view and the update function. Since the complete state is captured in the model and every state change is explicit by processing a message, this opens a whole new world of debugging features like [time travelling](https://fable-elmish.github.io/debugger/).  
Elmish 的程序抽象提供了“粘合”函数，例如 `mkSimple` 将这三个部分绑定到一个应用程序中。在内部，Elmish 为我们提供了一个消息循环，并负责在视图和更新函数之间调度消息。由于模型中捕获了完整的状态，并且通过处理消息来明确每个状态更改，因此这打开了时间旅行等调试功能的全新世界。

## Parent-child composition  
亲子作文

In the last section we saw how to build a very basic sample app in the MVU architecture. For this minimal example it's not very clear what the benefit is compared to the original implementation that was using mutation. So, in the following example we'll use the counter as a _module_ and create a _list of counters_.  
在上一节中，我们了解了如何在 MVU 架构中构建一个非常基本的示例应用程序。对于这个最小的示例，与使用突变的原始实现相比，并不清楚有什么好处。因此，在下面的示例中，我们将使用计数器作为模块并创建计数器列表。

### Composed Models 组合模型

As before let's start with the model:  
和之前一样，让我们​​从模型开始：

```fsharp
module CounterList
type Model = Counter.Model list

type Msg = 
| Insert
| Remove
| Modify of int * Counter.Msg

let init() : Model =
    [ Counter.init() ]
```

In this case we have a _list_ of counter models and a new message type. We can signal to:  
在本例中，我们有一个计数器模型列表和一个新消息类型。我们可以向以下人员发出信号：

- Insert a new counter  
    插入新计数器
- Remove the latest counter  
    删除最新的计数器
- Dispatch a counter message to the corresponding counter in the list.  
    向列表中相应的计数器发送计数器消息。

When the application starts we will start with one counter as provided by the `init` function. As you can see, every part of the model refers to the corresponding part of the submodel. `CounterList.Model` is a collection of `Counter.Model`, the `CounterList.Msg` uses the `Counter.Msg` as payload and the `CounterList.init` function calls the `Counter.init` function.  
当应用程序启动时，我们将从 `init` 函数提供的一个计数器开始。正如您所看到的，模型的每个部分都引用子模型的相应部分。 `CounterList.Model` 是 `Counter.Model` 的集合， `CounterList.Msg` 使用 `Counter.Msg` 作为负载， `CounterList.init` 函数调用 `Counter.init` 函数。

### Composed Views 组合视图

Let's take a look at the `View` code:  
我们看一下 `View` 代码：

```fsharp
let view model dispatch =
    let counters =
        model
        |> List.mapi (fun pos counterModel -> 
            Counter.view
                counterModel
                (fun msg -> dispatch (Modify (pos, msg)))) 

    div [] [ 
        yield button [ OnClick (fun _ -> dispatch Remove) ] [ "Remove" ]
        yield button [ OnClick (fun _ -> dispatch Insert) ] [ "Add" ]
        yield! counters ]
```

It's not that much different to the view code of the counter itself, except now we render a list of counters and wrap the messages with position information. This code may look a bit unfamiliar at this point, but the F# compiler is helping us here. There is only one way to get it to compile - and that's the correct way! Once you've done this step a few times, it becomes second nature - just like any task that you become familiar with.  
它与计数器本身的视图代码没有太大区别，只是现在我们渲染计数器列表并用位置信息包装消息。这段代码此时看起来可能有点陌生，但 F# 编译器在这里为我们提供了帮助。只有一种方法可以编译它 - 这就是正确的方法！一旦您完成此步骤几次，它就会成为第二天性 - 就像您熟悉的任何任务一样。

### Composed Updates 组合更新

As with the Model, we also see a nice symmetry in the Views. But what about the **Update** part?  
与模型一样，我们在视图中也看到了很好的对称性。但是更新部分呢？

```fsharp
let update (msg:Msg) (model:Model) =
    match msg with
    | Insert ->
        Counter.init() :: model // append to list
    | Remove ->
        match model with
        | [] -> []              // list is already empty
        | x :: rest -> rest     // remove from list
    | Modify (pos, counterMsg) ->
        model
        |> List.mapi (fun i counterModel ->
            if i = pos then
                Counter.update counterMsg counterModel
            else
                counterModel) }
```

Now this may come as no surprise, but we have the same situation here: `CounterList.update` forwards to `Counter.update` of the corresponding counter! We end up with something really beautiful, since now we have a `CounterList` component which exposes exactly the same elements as the `Counter` itself, namely **Model**, **View** and **Update**. This allows us to use the `CounterList` itself as a component!  
现在这可能并不奇怪，但我们这里有同样的情况： `CounterList.update` 转发到相应计数器的 `Counter.update` ！我们最终得到了一些非常漂亮的东西，因为现在我们有一个 `CounterList` 组件，它公开了与 `Counter` 本身完全相同的元素，即模型、视图和更新。这允许我们使用 `CounterList` 本身作为组件！

## F# - a great fit for the MVU architecture  
F# - 非常适合 MVU 架构

The MVU architecture was made famous by the Elm language, but can be used in many languages, even in vanilla [JavaScript](https://github.com/ccorcos/elmish). F# - like Elm - is a language in the ML family of programming language, and provides features such as pattern matching that are extremely powerful and particularly useful for modelling state machines. In conjunction with F# features such as [Discriminated Unions](https://fsharpforfunandprofit.com/posts/discriminated-unions/), this allows us to create applications in a type-safe manner that cater for all possibilities. The compiler provides us with guidance when corner cases are not dealt with, leading to quicker development time, whilst allowing F# developers to rapidly create UIs in a typesafe manner. The larger and more complex an application becomes, the greater the benefit; the F# compiler (like the Elm compiler) emits warnings for us when we forget to handle all possibilites - even for more complicated patterns - meaning less time spent debugging or writing unit tests, and more time delivering business value.  
MVU 架构因 Elm 语言而闻名，但可以在多种语言中使用，甚至可以在普通 JavaScript 中使用。 F# 与 Elm 一样，是 ML 编程语言系列中的一种语言，提供模式匹配等功能，这些功能非常强大，对于状态机建模特别有用。与受歧视联合等 F# 功能相结合，这使我们能够以类型安全的方式创建应用程序，以满足所有可能性。当未处理极端情况时，编译器为我们提供指导，从而缩短开发时间，同时允许 F# 开发人员以类型安全的方式快速创建 UI。应用程序变得越大、越复杂，好处就越大；当我们忘记处理所有可能性时（即使是更复杂的模式），F# 编译器（如 Elm 编译器）会向我们发出警告，这意味着花在调试或编写单元测试上的时间更少，而有更多时间提供业务价值。

## More Information 更多信息

For more information about Elmish and the MVU architecture please see the following resources:  
有关 Elmish 和 MVU 架构的更多信息，请参阅以下资源：

- "Modern app development with Fable and React Native" from NDC Oslo 2017 - [Video](https://www.youtube.com/watch?v=fmaPeUBWZuM)  
    “使用 Fable 和 React Native 进行现代应用程序开发”来自 NDC Oslo 2017 - 视频
- "The Elm Architecture" in the [elm docs](https://guide.elm-lang.org/architecture/)  
    elm 文档中的“Elm 架构”
- Elmish for F# [docs](https://fable-elmish.github.io/) F# 文档的 Elmish
- [Elmish for vanilla JavaScript  
    用于原生 JavaScript 的 Elmish](https://github.com/ccorcos/elmish)
- Fable compiler [docs](http://fable.io/) Fable 编译器文档