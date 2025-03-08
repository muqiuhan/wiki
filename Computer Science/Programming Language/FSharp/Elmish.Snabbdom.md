#fsharp #web

I've been recently playing with [Feliz.Engine](https://github.com/alfonsogarciacaro/Feliz.Engine/tree/main/samples/Feliz.Snabbdom), an attempt to take advantage of the great work done by Zaid Ajaj and contributors with [Feliz](https://zaid-ajaj.github.io/Feliz/) when writing non-React applications. As part of this I wanted to check how easy was to adapt Feliz.Engine to an alternative Virtual-DOM implementation, and I read good things about [Snabbdom](https://github.com/snabbdom/snabbdom) so I gave it a go. This started just as an experiment but I've been pleasantly surprised by how simple yet powerful Snabbdom is, and more importantly, how well it fits with the [Elmish architecture](https://elmish.github.io/), so I want to share with you my findings hoping that you find them useful.  
我最近一直在玩 Feliz.Engine，试图利用 Zaid Ajaj 和 Feliz 贡献者在编写非 React 应用程序时所做的出色工作。作为其中的一部分，我想检查将 Feliz.Engine 适应替代 Virtual-DOM 实现有多容易，并且我阅读了有关 Snabbdom 的好文章，所以我尝试了一下。一开始只是一个实验，但我对 Snabbdom 的简单而强大感到惊喜，更重要的是，它与 Elmish 架构的契合程度，所以我想与您分享我的发现，希望您发现它们有用。

There was recently a discussion in Twitter about the [problems with Fable Elmish](https://twitter.com/7sharp9_/status/1365270255170428928). So far, Elmish in Fable apps has always used React as the view engine, including React native for mobile (there are also Elmish implementations for non-Fable platforms like [WPF](https://github.com/elmish/Elmish.WPF), [Xamarin](https://fsprojects.github.io/Fabulous/) or [Blazor](https://fsbolero.io/docs/Elmish)), and there's been always friction between the concept of "component" in Elmish and React. This is a bit of technical discussion and I won't go into detail here, among other reasons because I've never managed to explain the difference in an understandable manner. Probably the easiest is to consider Elm/Elmish don't really have a notion of "component" as Dave Thomas explains. It's true the Fable Elmish community tends to "componentize" apps maybe under the influence of React, which sometimes leads to an excess of boilerplate to wire everything.  
最近 Twitter 上有一场关于《Fable Elmish》问题的讨论。到目前为止，《Fable》应用中的 Elmish 一直使用 React 作为视图引擎，包括适用于移动设备的 React Native（也有针对 WPF、Xamarin 或 Blazor 等非 Fable 平台的 Elmish 实现），并且“ Elmish 和 React 中的组件”。这是一些技术讨论，我不会在这里详细介绍，除其他原因外，因为我从未设法以可理解的方式解释其中的差异。也许最简单的方法是考虑 Elm/Elmish 并没有真正的“组件”概念，正如 Dave Thomas 所解释的那样。确实，Fable Elmish 社区可能在 React 的影响下倾向于“组件化”应用程序，这有时会导致过多的样板文件来连接所有内容。

It's possible to write an Elmish/React app with just a single view function, and some apps work well that way. But to take advantage of most of React features, like devtools, memoization or life-cycle events, you do need components, as React understands them. This is why some, myself guilty as charged, have been trying to drive towards more use of React components with Elmish. An important move for this has been the [`useElmish` React hook](https://zaid-ajaj.github.io/Feliz/#/Hooks/UseElmish) which many Fable devs have successfully adopted. But at this point Elmish gets reduced to manage the internal state of your components and your app gets eventually architected the React-way. This is not a bad thing if you already know React, but this post is about "rediscovering" the power of Elmish as I've been experiencing recently.  
可以仅使用单个视图函数编写 Elmish/React 应用程序，并且某些应用程序可以很好地工作。但要利用大多数 React 功能，例如开发工具、记忆或生命周期事件，您确实需要组件，正如 React 所理解的那样。这就是为什么一些人（我本人也有罪）一直在尝试推动更多地使用 Elmish 中的 React 组件。为此，一个重要的举措是 `useElmish` React hook，许多 Fable 开发人员已成功采用。但此时 Elmish 被简化为管理组件的内部状态，并且您的应用程序最终以 React 方式构建。如果您已经了解 React，这并不是一件坏事，但这篇文章是关于“重新发现”Elmish 的力量，正如我最近所经历的那样。

What if we try the other way around, that is, not worrying about "componentizing" our application? This is actually the original proposal of Elm/Elmish and what you get by using a low-level Virtual-DOM library like Snabbdom, instead of a full-fledged one like React. When I started trying to run Feliz.Engine with Snabbdom it was just about API ergonomics but being able to enjoy "pure" Elmish without giving up DOM control has been really freeing. Why I'm excited about Snabbdom? These are some of the reasons for it:  
如果我们尝试相反的方式，即不担心“组件化”我们的应用程序，会怎么样？这实际上是 Elm/Elmish 的最初提议，以及通过使用像 Snabbdom 这样的低级 Virtual-DOM 库而不是像 React 这样的成熟库所得到的结果。当我开始尝试使用 Snabbdom 运行 Feliz.Engine 时，它​​只是关于 API 人体工程学，但能够在不放弃 DOM 控制的情况下享受“纯粹的”Elmish 真的很自由。为什么我对 Snabbdom 感到兴奋？以下是一些原因：

#### It's just functions![](https://fable.io/blog/2021/2021-03-02-Announcing-Elmish-Snabbdom.html#its-just-functions) 这只是函数！#

There's no concept of component that clashes with Elmish, just composable functions from beginning to end. Again, you ca do the same with React but as soon as you need to deal with the DOM or some other features you need the components. This is not the case of Snabbdom, keep reading.  
没有与 Elmish 冲突的组件概念，只有从头到尾的可组合函数。同样，您可以对 React 执行相同的操作，但是一旦您需要处理 DOM 或其他一些功能，您就需要组件。 Snabbdom 的情况并非如此，请继续阅读。

#### CSS transitions built in[](https://fable.io/blog/2021/2021-03-02-Announcing-Elmish-Snabbdom.html#css-transitions-built-in)  
内置 CSS 过渡#

Easy CSS transitions was one of biggest [Svelte](https://svelte.dev/) appeals for me, and I was very surprised to see Snabbdom has a similar mechanism. Together with the wonderful Feliz API (check [the differences](https://github.com/alfonsogarciacaro/Feliz.Engine/blob/main/README.md) in Feliz.Engine), we can get a nice zoom-in/zoom-out effect just by attaching some styles to a node.  
简单的 CSS 转换是 Svelte 对我最大的吸引力之一，我很惊讶地看到 Snabbdom 也有类似的机制。结合精彩的 Feliz API（查看 Feliz.Engine 中的差异），我们只需将一些样式附加到节点即可获得不错的放大/缩小效果。

```
Html.li [
    Attr.className "box"

    Css.opacity 0.
    Css.transformScale 1.5
    // Snabbdom doesn't support `all`, we need to list all the transitioning properties
    Css.transitionProperty(transitionProperty.opacity, transitionProperty.transform)
    Css.transitionDurationSeconds 0.5
    Css.delayed [
        Css.opacity 1.
        Css.transformScale 1.
    ]
    Css.remove [
        Css.opacity 0.
        Css.transformScale 0.1
    ]
```

![Snabbdom CSS transitions](https://fable.io/static/img/blog/snabbdom-css-transitions.gif)

Learn more about Snabbdom CSS transitions [here](https://github.com/snabbdom/snabbdom#delayed-properties).  
在此处了解有关 Snabbdom CSS 过渡的更多信息。

#### Memoization[](https://fable.io/blog/2021/2021-03-02-Announcing-Elmish-Snabbdom.html#memoization) 记忆#

In theory, given that a pure Elmish app fully recreates the whole virtual DOM for every tiny change it's important to be able to skip the parts of your app that don't need to change (in reality, this usually is not a performance issue thankfully). But memoization has been one of the biggest pain-points when writing Fable/React bindings (still is). Because of nuances of how JS/F# languages work and the way React expects you to declare a memoized component. a [common pitfall](https://zaid-ajaj.github.io/Feliz/#/Feliz/React/CommonPitfalls) is to recreate the component for every function call rendering memoization useless. With Feliz.Snabbdom we just need to wrap a call with the `memoize` helper. For example, if we are displaying a list of Todos:  
理论上，考虑到纯 Elmish 应用程序会针对每一个微小的更改完全重新创建整个虚拟 DOM，因此能够跳过应用程序中不需要更改的部分非常重要（实际上，幸运的是，这通常不是性能问题） ）。但记忆一直是编写 Fable/React 绑定时最大的痛点之一（仍然是）。由于 JS/F# 语言工作方式的细微差别以及 React 希望您声明记忆组件的方式。一个常见的陷阱是为每个函数调用重新创建组件，从而使记忆变得无用。对于 Feliz.Snabbdom，我们只需要使用 `memoize` 帮助器来包装调用。例如，如果我们要显示待办事项列表：

```
let renderTodo dispatch (todo: Todo, editing: string option) = ...

let renderTodoList (state: State) (dispatch: Msg -> unit) =
    Html.ul (
        state.TodoList |> List.map (fun todo ->
            todo,
            state.Editing |> Option.bind (fun (i, e) -> if i = todo.Id then Some e else None))
        |> List.map (renderTodo dispatch)
    )
```

We just need to wrap the `renderTodo` call (here also provide a way to get a unique id from the arguments). Note that we don't need to check `dispatch` for the memoization, so we can just partially apply it before the wrapping:  
我们只需要包装 `renderTodo` 调用（这里还提供了一种从参数中获取唯一 id 的方法）。请注意，我们不需要检查 `dispatch` 的记忆化，因此我们可以在包装之前部分应用它：

```
let renderTodoList (state: State) (dispatch: Msg -> unit) =
    Html.ul (
        state.TodoList |> List.map (fun todo ->
            todo,
            state.Editing |> Option.bind (fun (i, e) -> if i = todo.Id then Some e else None))
        |> List.map (memoizeWithId (renderTodo dispatch) (fun (t, _) -> t.Id))
    )
```

#### Lifecycle hooks[](https://fable.io/blog/2021/2021-03-02-Announcing-Elmish-Snabbdom.html#lifecycle-hooks) 生命周期挂钩#

Unlike React ones, [hooks in Snabbdom](https://github.com/snabbdom/snabbdom#hooks) are very easy to understand. They are just events fired at different points of the lifecycle of a virtual node, as when they get inserted into or removed from the actual DOM. Very conveniently, the virtual node holding a reference to the actual DOM element is passed as argument to the event handler so it's easy for example to get the actual height of an element.  
与 React 不同，Snabbdom 中的钩子非常容易理解。它们只是在虚拟节点生命周期的不同点触发的事件，就像它们插入实际 DOM 或从实际 DOM 中删除时一样。非常方便的是，保存对实际 DOM 元素的引用的虚拟节点作为参数传递给事件处理程序，因此可以轻松获取元素的实际高度。

React hooks allow you to do similar things, but they're designed in a way that forces you to translate your thinking into the React way of doing things. Let's say you want to turn some text into an input on double click, then select all the text and attach an event to the document so if you click outside the containing box you cancel the edit. For this, in React you need to (forgive me if there's a more clever way of doing this that I'm missing):  
React hooks 允许你做类似的事情，但它们的设计方式迫使你将你的想法转化为 React 的做事方式。假设您想通过双击将某些文本转换为输入，然后选择所有文本并将事件附加到文档，这样如果您在包含框之外单击，则会取消编辑。为此，在 React 中你需要（如果我缺少更聪明的方法来做到这一点，请原谅我）：

1. Make sure the function you are in is a component because this is required to use hooks.  
    确保您所在的函数是一个组件，因为这是使用钩子所必需的。
2. Declare a reference to hold the actual input element with `useRef` hook (beware! you don't have the actual element yet).  
    使用 `useRef` 钩子声明一个引用来保存实际的输入元素（注意！您还没有实际的元素）。
3. Pass the value returned by `useRef` to a `ref` prop on the input element so React fills it.  
    将 `useRef` 返回的值传递给输入元素上的 `ref` 属性，以便 React 填充它。
4. Declare an effect with `useEffect` hook. Because you want the effect to happen when the input appears, you need to pass an array with a flag like `isEditable`.  
    使用 `useEffect` 钩子声明效果。因为您希望在输入出现时发生效果，所以您需要传递一个带有 `isEditable` 等标志的数组。
5. The effect will happen when `isEditable` changes from false to true or from true to false, so make sure `isEditable` is true before running the effect.  
    当 `isEditable` 从 false 变为 true 或从 true 变为 false 时，效果就会发生，因此在运行效果之前请确保 `isEditable` 为 true。
6. Now get the input element from the value you declared in 2. Select the text and attach the event to the document body, return a disposable function to detach the event when `isEditable` changes to false.  
    现在从 2 中声明的值获取输入元素。选择文本并将事件附加到文档正文，返回一次性函数以在 `isEditable` 更改为 false 时分离事件。

On the other hand, in Snabbdom if you want to, when an input element appears, select all the text, attach an event to the html body and detach it when the input disappears you need to:  
另一方面，在 Snabbdom 中，如果您愿意，当输入元素出现时，选择所有文本，将事件附加到 html 正文，并在输入消失时将其分离，您需要：

1. Add an `insert` hook to the input, so when it appears, you can select all the text, attach an event to the html body and return a disposable to detach it when the input disappears.  
    在输入中添加一个 `insert` 钩子，这样当它出现时，您可以选择所有文本，将事件附加到 html 主体，并返回一个一次性事件，以便在输入消失时将其分离。

Well, I'm cheating a bit here, in "raw" Snabbdom keeping a reference to the disposable and disposing it when the element is destroyed is slightly more contrived, but luckily Feliz.Snabbdom provides an overload to `Hook.insert` so this is automatically done for you if the callback returns a disposable:  
好吧，我在这里有点作弊，在“原始”Snabbdom 中保留对一次性的引用并在元素被销毁时处理它，这有点做作，但幸运的是 Feliz.Snabbdom 为 `Hook.insert` 提供了重载因此，如果回调返回一次性值，则会自动为您完成此操作：

```
Html.input [
    Attr.classes [ "input"; "is-medium" ]
    Attr.value editing
    Ev.onTextChange (SetEditedDescription >> dispatch)
    onEnterOrEscape dispatch ApplyEdit CancelEdit

    Hook.insert(fun vnode ->
        let el = vnode.elm.AsInputEl
        el.select() // Select all text

        let parentBox = findParentWithClass "box" el
        // This function attachs the event to the body
        // and returns a disposable to detach it
        BodyEv.onMouseDown(fun ev ->
            if not (parentBox.contains(ev.target :?> _)) then
                CancelEdit |> dispatch)
    )
]
```

> Did you notice `BodyEv.onMouseDown`? This is another nice use-case of [Feliz.Engine abstract classes](https://github.com/alfonsogarciacaro/Feliz.Engine/blob/cbf4b90de929d7202f941ef091436a8845634b80/src/Feliz.Snabbdom/Feliz.Snabbdom.fs#L163-L168), it implements `EventEngine` by making it return a disposable.  
> 你注意到 `BodyEv.onMouseDown` 了吗？这是 Feliz.Engine 抽象类的另一个很好的用例，它通过返回一次性对象来实现 `EventEngine` 。

  

So Snabbdom is great, now what? Does this mean you need to ditch React for Fable apps? Of course not! React is still a great choice, with many useful tools and a gigantic ecosystem. It's true there are frictions with Elmish but thanks to the work of Zaid, Maxime Mangel and many others, together with the `ReactComponent` plugin in Fable 3 they've become more bearable. So if you already know React quirks and/or rely on some of its tools and libraries you can be sure will still be well supported by Fable. Just if you're mainly interested in Elmish and don't really care for the underlying renderer you may want to give Elmish.Snabbdom a try if you're looking for less complexity. Clone the repo and try out [this sample](https://github.com/alfonsogarciacaro/Feliz.Engine/tree/main/samples/Feliz.Snabbdom) to see how Elmish.Snabbdom can work for you.  
所以 Snabbdom 很棒，现在怎么办？这是否意味着您需要放弃 React 而使用 Fable 应用程序？当然不是！ React 仍然是一个不错的选择，拥有许多有用的工具和庞大的生态系统。确实与 Elmish 存在摩擦，但由于 Zaid、Maxime Mangel 和许多其他人的工作，再加上《神鬼寓言 3》中的 `ReactComponent` 插件，这些摩擦已经变得更容易忍受。因此，如果您已经了解 React 的怪癖和/或依赖它的一些工具和库，您可以肯定 Fable 仍然会提供良好的支持。如果您主要对 Elmish 感兴趣并且并不真正关心底层渲染器，如果您正在寻找较低的复杂性，您可能想尝试一下 Elmish.Snabbdom。克隆存储库并尝试此示例，看看 Elmish.Snabbdom 如何为您工作。

And! If you are really into a purer Fable/F# experience and want more control of the DOM, take also a look at the awesome work of David Dawkins with [Sutil](https://davedawkins.github.io/Sutil)!  
和！如果您确实喜欢更纯粹的 Fable/F# 体验并希望更多地控制 DOM，还可以看看 David Dawkins 与 Sutil 的精彩作品！