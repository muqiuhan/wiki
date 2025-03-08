#fsharp #fp #software-engineering 

_his article is intended for people who are already familiar with the MVU architecture. You can learn more about it on the_ [_Elmish website_](https://elmish.github.io/elmish/)_._  
本文面向已经熟悉 MVU 架构的人员。您可以在 Elmish 网站上了解更多相关信息。

I am one of the maintainer of Elmish and I created several [Fable](http://fable.io) / Elmish libraries and applications. When doing so, I needed to understand Elmish deeply and I want to take the opportunity of [_F# Advent Calendar_](https://sergeytihon.com/2018/10/22/f-advent-calendar-in-english-2018/) to share with you my tips and knowledge.  
我是 Elmish 的维护者之一，我创建了几个 Fable / Elmish 库和应用程序。这样做时，我需要深入了解 Elmish，我想利用 F# Advent Calendar 的机会与大家分享我的技巧和知识。

I will be covering a lot in this article, so grab yourself a cup of tea 🍵, take a deep breath 💨 and let’s get started 🏁.  
我将在这篇文章中介绍很多内容，所以给自己喝杯茶🍵，深呼吸💨，让我们开始吧🏁。

# Commands 命令

A command is a container for a function that Elmish executes immediately but, may schedule dispatch of message at any time.  
命令是 Elmish 立即执行的函数的容器，但可以随时安排消息的发送。

For example: 例如：

- Issue immediately a new Message  
    立即发出新消息
- Make an HTTP call and then return the result in your application via a Message  
    进行 HTTP 调用，然后通过消息将结果返回到您的应用程序中
- Save data in your storage  
    将数据保存在您的存储中

## Basic usage 基本用法

In this section, I will cover all the default functions offered by Elmish and show an example of it.  
在本节中，我将介绍 Elmish 提供的所有默认函数并展示它的示例。

Use `Cmd.none` to schedule no Commands.  
使用 `Cmd.none` 不安排任何命令。

```F#
type Model =
    { Tick : int }
    
type Msg =
    | Tick
    
let private update msg model =
    match msg with
    | Tick ->
        { model with Tick = model.Tick + 1 }, Cmd.none
```

Use `Cmd.ofMsg` to schedules another message directly, it can be seen as a way to chain messages.  
使用 `Cmd.ofMsg` 直接调度另一条消息，可以看作是一种链接消息的方式。
```F#
type Model =
    { Value : string }

type Msg =
    | ChangeValue of string
    | ValidateData
    
let private update msg model =
    match msg with
    | ChangeValue newValue ->
        { model with Value = newValue }, Cmd.ofMsg ValidateData
```
Use`Cmd.ofAsync` to evaluate an `async` block and map the result into success or error (of exception).  
使用 `Cmd.ofAsync` 评估 `async` 块并将结果映射为成功或错误（异常）。
```F#
type Model =
    { InputValue : string
      UpperValue : string }
      
type Msg =
    | ToUpper of string
    | OnUpperResult of string
    | OnUpperError of exn

let private update (msg : Msg) (model : Model) =
    match msg with
    | ToUpper txt ->
        let asyncUpper (txt : string) =
            async {
                do! Async.Sleep 1000

                return txt.ToUpper()
            }

        model, Cmd.ofAsync asyncUpper txt OnUpperResult OnUpperError

    | OnUpperResult result ->
        { model with UpperValue = result }, Cmd.none

    | OnUpperError error ->
        Browser.console.error error
        model, Cmd.none
```
Use `Cmd.ofFunc` to evaluate a simple function and map the result into success or error (of exception).  
使用 `Cmd.ofFunc` 评估一个简单的函数并将结果映射为成功或错误（异常）。
```F#
type Model =
    { CurrentRoute : Route
      Value : string
      Tick : int
      UpperValue : string }

type Msg =
    | Save of string
    | OnSaveSuccess of bool
    | OnSaveError of exn
    
let private update (msg : Msg) (model : Model) =
    match msg with
    | Save txt ->
        let save (txt : string) =         
            // If there is an error, an exception will be thrown and captured by Cmd.ofFunc
            Browser.localStorage.setItem("my-app.input", txt)
            true

        model, Cmd.ofFunc save txt OnSaveSuccess OnSaveError

    | OnSaveSuccess result ->
        // Here we can notify the user that the save succeeded
        model, Cmd.none

    | OnSaveError error ->
        // Here we can notifu the user that the save failed
        Browser.console.error error
        model, Cmd.none
```

Use`Cmd.performFunc` to evaluate a simple function and map the success to a message discarding any possible error.  
使用 `Cmd.performFunc` 评估一个简单的函数，并将成功映射到一条消息，丢弃任何可能的错误。
```F#
type Model =
    { IsLoading : bool
      Value : string }

type Msg =
    | Load
    | OnLoadSuccess of bool
    
let private update (msg : Msg) (model : Model) =
    match msg with
    | Load ->
        // This function can never fail so we can use Cmd.performFunc
        let load () =
            let storedValue : string = Browser.localStorage.getItem("my-app.input") :?> string
            if isNull storedValue then
                ""
            else
                storedValue

        { model with IsLoading = true }, Cmd.performFunc load () OnLoadSuccess

    | OnLoadSuccess value ->
        { model with IsLoading = false
                     Value = value }, Cmd.none
```

Use`Cmd.attemptFunc` to evaluate a simple function and map the error (in case of exception)  
使用 `Cmd.attemptFunc` 计算一个简单的函数并映射错误（如果出现异常）
```F#
type Model =
    { Value : string }

type Msg =
    | ChangeValue of string
    | OnLogError of exn
    
let private update (msg : Msg) (model : Model) =
    match msg with
    | ChangeValue newValue ->
        let log (msg : string) =
            // Send a msg to your logger
            // If there is an error it will throw
            Log.send msg

        let msg = sprintf "Value changed from %s to %s" model.Value newValue        

        { model with Value = newValue }, Cmd.attemptFunc log msg OnLogError

    | OnLogError value ->
        // There was an error during logging, should we do something ?
        model, Cmd.none
```
Use `Cmd.ofPromise` to call `promise` block and map the results.  
使用 `Cmd.ofPromise` 调用 `promise` 块并映射结果。
```F#
type Model =
    { Value : int }

type Msg =
    | Submit
    | OnPromiseSuccess of int
    | OnPromiseError of exn
    
let private update (msg : Msg) (model : Model) =
    match msg with
    | Submit ->
        let myPromise () =
            promise {
                do! Promise.sleep 1000
                return 10
            }

        model, Cmd.ofPromise myPromise () OnPromiseSuccess OnPromiseError

    | OnPromiseSuccess value ->
        { model with Value = value }, Cmd.none

    | OnPromiseError error ->
        Browser.console.error error
        model, Cmd.none
```

Use `Cmd.ofSub` to call the subscriber. This is useful when you are dealing with an API which use callbacks or to listen to an event.  
使用 `Cmd.ofSub` 呼叫订阅者。当您处理使用回调或侦听事件的 API 时，这非常有用。

```F#
type Model =
    { Position : Browser.Position option
      ErrorMessage : string }

type Msg =
    | GetPosition
    | GetPositionSuccess of Browser.Position
    | GetPositionError of Browser.PositionError

module Sub =
    // Subscriber used to reach gealocation API
    let getPosition onSuccess onError dispatch =
        Browser.navigator.geolocation.getCurrentPosition(
          onSuccess >> dispatch,
          onError >> dispatch
        )

let update (msg:Msg) (model:Model) =
    match msg with
    | GetPosition ->
        model, Cmd.ofSub (Sub.getPosition GetPositionSuccess GetPositionError)
    
    | GetPositionSuccess posision ->
        { model with Position = Some position
                     ErrorMessage = "" }, Cmd.none
    
    | GetPositionError error ->
        { model with Position = None
                     ErrorMessage = error.message }, Cmd.none
```

Use `Cmd.batch` to aggregate multiple commands. You can aggregate any of the previous commands together.  
使用 `Cmd.batch` 聚合多个命令。您可以将前面的任何命令聚合在一起。

```F#
type Model =
    { Data1 : int
      Data2 : int }

type Msg =
    | FetchDatas
    | OnFetchData1 of int
    | OnFetchData2 of int
    | OnPromiseError of exn
    | OnLogError of exn
    
let private update (msg : Msg) (model : Model) =
    match msg with
    | FetchDatas ->
        model, Cmd.batch [ Cmd.ofPromise fetchData1 () OnFetchData1 OnPromiseError 
                           Cmd.ofPromise fetchData2 () OnFetchData2 OnPromiseError 
                           Cmd.performFunc log "FetchDatas message has been treated" OnLogError ]

    | OnFetchData1 data1 ->
        { model with Data1 = data1 }, Cmd.performFunc log "FetchData1 data has been received" OnLogError

    | OnFetchData2 data2 ->
        { model with Data2 = data2 }, Cmd.performFunc log "FetchData2 data has been received" OnLogError

    | OnPromiseError error ->
        Browser.console.error("An error occured when fetching data", error)
        model, Cmd.none

    | OnLogError error ->
        Browser.console.error("An error occured when logging", error)
        model, Cmd.none
```

The functions provided out of the box by Elmish are enough to cover most of the needs of your application. However, when working on a library or with a specific API you could want to create your own commands. We will take a look at it later.  
Elmish 提供的开箱即用的功能足以满足您应用程序的大部分需求。但是，当使用库或特定 API 时，您可能想要创建自己的命令。我们稍后会看一下。

# Modelize your Model according to your needs  
根据您的需求对模型进行建模

When working with a Single Page Application, you will have to deal with navigation and need to reflect it in your models.  
使用单页应用程序时，您将必须处理导航并需要将其反映在模型中。

In general, when I am working with at a page level, I use Discrimination Union so I can store only the active page state.  
一般来说，当我在页面级别工作时，我使用歧视联盟，这样我就可以只存储活动页面状态。  
For example, if in your application you have a **login** page **you can’t store** your **authenticated** page state in your application because you can’t fetch the resource to display it.  
例如，如果您的应用程序中有一个登录页面，您无法在应用程序中存储经过身份验证的页面状态，因为您无法获取资源来显示它。

In general my main `Model` looks like this:  
一般来说，我的主要 `Model` 看起来像这样：
```F#

[<RequireQualifiedAccess>]
type AuthPage =
    // Section for the website related to Messages management
    // For example
    // - Display the list of Messages
    // - Create a message
    | Messages of Messages.Model
    // Administration section of the website
    | Administration of Administration.Model

[<RequireQualifiedAccess>]
type Page =
    // The application is loading, display a loader
    | Loading
    // Login page
    | Login of Login.Model
    // From here, the user need to be authenticated
    | AuthPage of AuthPage
    // Global error page like:
    // - Generic error
    // - You need to be connected
    | Errored of Errored.Reason
    // A page that can be accessed when the user is not logged in
    // For example, to reset it's password from an URL
    | ResetAccount of ResetAccount.Model

type Model =
    { // Store the user session information
      // If Some xxx, then the user is signed-in
      // If None, the user isn't connected
      Session : User option
      // Store the active page state
      ActivePage : Page
      // Store the current route information, this is useful if your
      // route have parameters than you need to pass to child later
      CurrentRoute : Router.Route option }```

Using DUs to represent your pages state helps you isolate your logics. For me, the biggest benefit compared to storing all page states in a big record, is that you are not caching your page when navigating.  
使用 DU 表示页面状态可以帮助您隔离逻辑。对我来说，与将所有页面状态存储在大记录中相比，最大的好处是您在导航时不会缓存页面。

Of course sometimes your application needs to maintain state between navigation. For example, if you have a form in several steps you could modelize it like this:  
当然，有时您的应用程序需要在导航之间维护状态。例如，如果您有一个分几个步骤的表单，您可以像这样对其进行建模：
```F#
type Rank =
    | One
    | Two

type Step1 =
    { Login : string
      Password : string }

type Step2 =
    { Age : int 
      Surname : string 
      Firstname : string }

type Model =
    { CurrentRank : Rank
      Step1 : Step1
      Step2 : Step2 }

let private view (model : Model) (dispatch : Msg -> unit) =
    match model.CurrentRank with
    | Rank.One -> // render step n°1 form
    | Rank.Two -> // render step n°2 form
```
# Everything is a function 一切都是函数

One thing we tend to forget when working with Elmish is that **everything is a function**. The main reason for this forgetfulness is that most of the examples only show you this code:  
使用 Elmish 时我们容易忘记的一件事是一切都是函数。造成这种遗忘的主要原因是大多数示例仅向您展示以下代码：
```F#
type Model =
    { Value : int }

type Msg =
    | SomeAction

let init =
    { Value = 0 }, Cmd.none

let update (msg : Msg) (model : Model) : Model * Cmd<Msg> =
    match msg with
    | SomeAction ->
        model, Cmd.none

let view (model : Model) (dispatch : Msg -> unit) : React.ReactElement =
    div [ ]
        [ str "This is my view" ]
```
So in our mind `update` **takes two arguments** when in fact we should think `update` **takes at least two arguments**. The same applies to `view` and `init` functions.  
因此，在我们看来 `update` 需要两个参数，而实际上我们应该认为 `update` 至少需要两个参数。这同样适用于 `view` 和 `init` 函数。

This idea is linked to the fact that a component’s `Model` should have all the information needed by the component, but this is not always true.  
这个想法与组件的 `Model` 应该拥有组件所需的所有信息有关，但这并不总是正确的。

## Pass data as an argument  
将数据作为参数传递

For example, if you have a session in your application in order to make HTTP calls, should you store this session in **each component’s** `Model` ?  
例如，如果您的应用程序中有一个会话用于进行 HTTP 调用，您是否应该将此会话存储在每个组件的 `Model` 中？  
**No**, you should store it at **one place** and then pass your session to the functions that need it.  
不，您应该将其存储在一个位置，然后将会话传递给需要它的函数。

The next code illustrates this situation, we request a `session` argument in our init function because we need to make an authenticated http request using `Http.Auth.*` (custom module used, includes the session info in a request).  
下一个代码说明了这种情况，我们在 init 函数中请求 `session` 参数，因为我们需要使用 `Http.Auth.*` 发出经过身份验证的 http 请求（使用的自定义模块，包括会话信息要求）。

```F#
open Elmish
open Fable.Import
open Fable.PowerPack

type Session = 
    { Token : string
      UserId : int }

type Model =
    | Loading
    | Loaded of string
    | Errored

type Msg =
    | OnFetchDataSuccess of string
    | OnFetchDataError of exn

let private fetchData (session : Session) : JS.Promise<string> =
    promise {
        let! res = Http.Auth.postRecord "/api/get-data" session
        return! res.text()
    }

let init (session : Session) =
    Loading, Cmd.ofPromise (fetchData session) OnFetchDataSuccess OnFetchDataError

let private update (msg : Msg) (model : Model) =
    match msg with
    | OnFetchDataSuccess value ->
        Loaded value, Cmd.none

    | OnFetchDataError error ->
        Browser.console.error error
        Errored, Cmd.none
```

## Pass a record as an argument  
将记录作为参数传递

If your view function takes several arguments you can use a record as an argument. It will force you to name the arguments and by doing so make your code easier to read and maintain over time.  
如果您的视图函数采用多个参数，您可以使用记录作为参数。它将迫使您命名参数，并通过这样做使您的代码随着时间的推移更易于阅读和维护。

```F#
type private SectionProps =
    { Icon : Fa.IconOption
      Count : int
      Label : string
      IsActive : bool
      ZoneColor : string }
  
let private renderSection (props : SectionProps) =
    // Render the view
    
let view (model : Model) =
    renderSection 
        { Icon = Fa.Solid.Cog
          Count = model.ZoneA.Value + model.ZoneB.Value
          Label = model.Zone.Name
          IsActive = model.IsActive
          ZoneColor = findColor model.Zone.ColorIndex }
```

It will also make it easier to optimize your code for react using `memoBuilder`.  
它还将使使用 `memoBuilder` 更轻松地优化代码以进行反应
```F#
type private SectionProps =
    { Icon : Fa.IconOption
      Count : int
      Label : string
      IsActive : bool
      ZoneColor : string }
  
let private renderSection = 
    memoBuilder "Section" (fun (props : SectionProps) ->
        // Render the view
    )
        
let view (model : Model) =
    renderSection 
        { Icon = Fa.Solid.Cog
          Count = model.ZoneA.Value + model.ZoneB.Value
          Label = model.Zone.Name
          IsActive = model.IsActive
          ZoneColor = findColor model.Zone.ColorIndex }
```

## Use helper function to work with your Domain  
使用辅助函数来处理您的域

For example, when an elmish component needs to fetch data from the server often, I like to show a loading animation. Here is how I handle it:  
例如，当一个精灵组件需要经常从服务器获取数据时，我喜欢显示加载动画。我是这样处理的：