#fsharp #fp #web #network #dotnet 
Hi,

I try to combine a WebRTC and Fable for some time in my daily work. I want to describe the process of implementing a signaling mechanism for it.

But at first, what is a WebRTC? [WebRTC](https://webrtc.org/) is a technology that allows exchanging data between N peers via UDP or TCP protocols. Exchange means that there is no server in the middle, but peers talk to each other (except situation when Relay connection is used). WebRTC is a native API available in all modern browsers. One of the things that are not ready by default is the signaling process. About which will be this article.

Before we go into details of implementation. Let’s explain some basic concepts around signaling. To establish a connection between 2 Peers they need to exchange messages with [SDP](https://developer.mozilla.org/en-US/docs/Glossary/SDP) (Session description protocol) information. The number of those messages depends on:

- the number of STUN/TURN server,
- type of connection (P2P/Relay).

SDP contains a lot of information needed to start a connection. Messages that would be exchanged are called Offer, Answer, and Candidates. We gonna use a [Trickle-Ice](https://webrtcglossary.com/trickle-ice/) mechanism. It means that the offer, answer, and candidates will be sent separately in an async manner. Candidates have information about “how to connect to a peer which create them” (more about them [here](https://developer.mozilla.org/en-US/docs/Web/API/RTCIceCandidate)). There is also a simpler way of establishing a connection where Answer and Offer would contain all needed candidates. But I want to focus on an async version because we are using it in our product because of better performance.

Switching messages as described above are named “signaling”. When I was looking for some ready to go solution for signaling I find out [Ably.io](https://www.ably.io/). I decided to give it a try in case of my base application, which should send a file from one peer to another.

When I was reading the Ably.io documentation. I realized that the only functions that I will need would be `publish` and `subscribe`. Which are used in a message channel context. Because of that, I created an Ably interface in F# which is a port of JS interfaces from their official lib.

```fsharp
module AblyMapping =
  type IMessage =
    abstract name: string
    abstract data: string

  type IChannel =
    abstract publish: string * string -> unit
    abstract subscribe: string * (IMessage -> unit) -> unit

  type IChannels =
    abstract ``get``: string -> IChannel

  type IRealTime =
    abstract channels: IChannels

  type IAbly =
    abstract Realtime: (string) -> IRealTime

  let Ably: IAbly = importAll "Ably"
```

Changes in packages.json.

```javascript
{
  …
  ”ably”: "^1.2.4"
}
```

I believe that the above code is self-explanatory when we look at the JS example from the Ably site.

```javascript
var ably = new Ably.Realtime('apikey');
var channel = ably.channels.get('gut-tub');

// Publish a message to the gut-tub channel
channel.publish('greeting', 'hello');

var ably = new Ably.Realtime('apikey');
var channel = ably.channels.get('gut-tub');

// Subscribe to messages on channel
channel.subscribe('greeting', function(message) {
  alert(message.data);
});
```

Initialization of ably could be found in `init` method.

```fsharp
let init (): Model * Cmd<Msg> =
  let channel =
    AblyMapping.Ably.Realtime “api-key”
    |> fun x -> x.channels.get “channel name"
  let model = {
    Role = None
    Channel = channel
    ConnectionState = WebRTC.ConnectionState.NotConnected
    WaitingCandidates = [] }
  model, Cmd.none
```

Right now, we could go to sending and receiving messages via Ably, but I start with installing the [WebRTC-adapter](https://www.npmjs.com/package/webrtc-adapter) library. This library gives me confidence that what I want to achieve with native API would work the same way in every browser. Since there are some slight differences between implementations.

I add the following line to the packages.json with the `webrtc-adapter` library.

```javascript
{
  …
  "webrtc-adapter": "^7.7.0"
}
```

Usage in F# code.

```fsharp
module WebRTC =
  let private adapter: obj = importAll "webrtc-adapter"
```

Thanks to the above. I’m sure that the interfaces are compatible across all supported browsers. We could switch to the code which is responsible for handling WebRTC. The code snippet is big because we need to distinguish the `sender` and `receiver` of a file. It is because we don’t have an application on the server-side. I started with the creation of a `PeerConnection` object with a communication server address.

```fsharp
let create () =
  let conf =
    [| RTCIceServer.Create( [| "turn:location:443" |], "user", "pass", RTCIceCredentialType.Password ) |]
    |> RTCConfiguration.Create
    |> fun x ->
      x.iceTransportPolicy <- Some RTCIceTransportPolicy.Relay
      x

    let pc = RTCPeerConnection.Create(conf)

    { Peer = pc; Channel = None }
```

Because I test everything locally, I force using a Relay connection. Otherwise, communication would be done via a local network. That would result in skipping the signaling phase (host candidates would be used). Worth mentioning is that using public STUN/TURN servers is not recommended. We are not sure how the configuration looks like and who owns them.

We create a `LifecycleModel` object which contains a `PeerConnection` and `RTCDataChannel` inside. And create an instance of it. It is created as a mutable field instead of part of the Elmish model. It is because of simplicity (default Thoth serializer couldn’t handle serializing `PeerConnection` also passing the whole big object wouldn’t be something that we want to do).

```fsharp
type LifecycleModel = {
  Peer: RTCPeerConnection
  Channel: RTCDataChannel option
}

...

let mutable peer: LifecycleModel = Unchecked.defaultof<_>

...

peer <- WebRTC.create ()
```

After the creation of the `PeerConnection` object. We see that in the UI we have a possibility to connect as `sender` or `receiver`.

![initialize](https://mnie.github.com/img/2021_01_05_webrtc_ably/initialize.png)

It would result in a distinction of how WebRTC is working and a different value for a `Role` field in a model. When we look at how the `sender` logic is implemented we start with the creation of `RTCDataChannel`. It would be used as a transfer channel between users. The creation of a channel is located in the `createChannel` function.

```fsharp
let createChannel pc name =
  let channel = pc.Peer.createDataChannel name
  { pc with Channel = Some channel}
```

Going to the configuration of messages send and received via `Ably`. When we are a `sender` we want to listen to `Answer` and `Candidate`(from the receiver) messages. How it is achieved is visible in below code snippet.

```fsharp
let subs =
  [
      "answer", fun (msg: AblyMapping.IMessage) -> dispatch (Msg.WebRTC (WebRTC.Msg.Answer msg.data))
      "receiver-candidate", fun (msg: AblyMapping.IMessage) -> dispatch (Msg.WebRTC (WebRTC.Msg.Candidate msg.data))
  ] |> Map.ofList
```

And combination with `Ably` channel is visible in `init` method from `Signaling` module.

```fsharp
let init (channel: IChannel) (subscribers: Map<string, IMessage -> unit>) =
  subscribers
  |> Map.iter (fun subscriberKey subscriberFunc ->
      channel.subscribe (subscriberKey, subscriberFunc)
  )
  channel
```

As we could see we subscribe to all messages with the given keys `answer` and `receiver-candidate`. When they occur we propagate Elmish messages that would be handle in the `update` method.

In the `receiver` scenario the difference is that we don’t create `RTCDataChannel` (this is why it is marked as `option` in `LifecycleModel`). We gather it when the connection would be in a `connecting` phase. If it would be created on its own we would receive an invalid state error. This is why we only subscribe to messages `Offer` and `Candidate`(from Sender).

When a subscription to Ably messages is ready we send an Elmish message `Signaling` with a ready channel which is updated in the application model. Whereas the WebRTC configuration is updated with callbacks to functions that need to be handled in a connection/data transfer process.

Assignment of those callbacks and how they are handled is visible in a `subscribe` function. Which should:

- Initialize `RTCDataChannel` through which data (file) would be transferred.

```fsharp
let private initReceiverChannel (pc: RTCPeerConnection) msgHandler dispatch =
  let mutable updatedChannel: RTCDataChannel = Unchecked.defaultof<_>
  let callback (ev: RTCDataChannelEvent) =
    let receiveChannel = subscribeChannel ev.channel msgHandler dispatch
    internalLog (sprintf "updating channel: %A" ev.channel.id)
    peer <- { peer with Channel = Some receiveChannel }

  pc.ondatachannel <- callback
  updatedChannel

let updatedChannel =
  if role = Role.Sender then
    match channel with
    | Some c ->
      internalLog (sprintf "initialize channel for: %A" role)
      subscribeChannel c msgHandler dispatch
    | None -> failwith "Channel is not initilized for sender"
  else
      internalLog (sprintf "initialize channel2 for: %A" role)
      initReceiverChannel pc msgHandler dispatch
```

- Handling connection state (only for diagnostic purposes).

```fsharp
pc.oniceconnectionstatechange <-
  fun _ ->
    internalLog (sprintf "Connection state changed to: %A" pc.iceConnectionState)
```

- Handling exchange of candidates.

```fsharp
pc.onicecandidate <-
  fun e ->
    match e.candidate with
    | None -> internalLog "Trickle ICE Completed"
    | Some cand ->
      cand.toJSON()
      |> Base64.``to``
      |> Signaling.WebRTCCandidate.Candidate
      |>
        if role = Role.Sender then
            Signaling.Notification.SenderCandidate
        else Signaling.Notification.ReceiverCandidate
      |> onSignal
```

Configuration of WebRTC is ready on the `sender` and `receiver` side.

![connect](https://mnie.github.com/img/2021_01_05_webrtc_ably/connect.png)

We could initiate a connection by clicking the `Connect` button. After clicking it the `init` method from the `WebRTC` module would be called.

```fsharp
let init connection onSignal =
  let pc, channel = connection.Peer, connection.Channel
  pc.createOffer().``then``(fun desc ->
    pc.setLocalDescription (desc) |> Promise.start
    if isNull desc.sdp then
      internalLog "Local description is empty"
    else
      desc
      |> Base64.``to``
      |> Signaling.WebRTCOffer.Offer
      |> Signaling.Notification.Offer
      |> onSignal)

  |> Promise.catch (sprintf "On negotation needed return error: %A" >> internalLog)
  |> Promise.start

  { Peer = pc
    Channel = channel }
```

We send `Offer` via Ably channel and set it as `LocalDescription` via `setLocalDescription` method on `PeerConnection` object. Right now the application flow is not visible when we look at the code at first. The other side of communication should receive `Offer` via Ably channel which would be then propagated as `Offer` Elmish message and handle in `update` method.

```fsharp
let setOffer (lifecycle: LifecycleModel) onSignal remote =
  try
    let desc = rtcSessionDescription remote
    internalLog (sprintf "setting offer: %A" desc)
    lifecycle.Peer.setRemoteDescription desc
    |> Promise.catch (sprintf "Failed to set remote description: %A" >> internalLog)
    |> Promise.start
    lifecycle.Peer.createAnswer().``then``(fun desc ->
      lifecycle.Peer.setLocalDescription (desc) |> Promise.start
      if isNull desc.sdp then
        internalLog "Local description is empty"
      else
        internalLog (sprintf "sending answer: %A" desc)
        desc
        |> Base64.``to``
        |> Signaling.WebRTCAnswer.Answer
        |> Signaling.Notification.Answer
        |> onSignal)

    |> Promise.catch (sprintf "On negotation needed errored with: %A" >> internalLog)
    |> Promise.start
  with e ->
    internalLog (sprintf "Error occured while adding remote description: %A" e)
```

It should set `Offer` from a `sender` as a `RemoteDescription` on a `receiver` side and in case of success generate `Answer`. It would be sent to the `sender` via Ably. The important thing to mention here is that after the creation of `Offer`/`Answer` `Candidates` are generated. They shouldn’t be set on the `PeerConnection` object before setting `Local` and `Remote` descriptions. Because of that Elmish model has a buffer for `Candidates` that could be received before setting `Remote`/`Local` descriptions.

Buffor handling:

```fsharp
if model.WaitingCandidates.Length > 0 then
  model.WaitingCandidates
  |> List.iter (WebRTC.setCandidate peer )
  { model with
              ConnectionState = WebRTC.ConnectionState.Connecting
              WaitingCandidates = [] }, Cmd.none
else
  { model with ConnectionState = WebRTC.ConnectionState.Connecting }, Cmd.none
```

Handling a message that contains `Candidates` looks like that.

```fsharp
if model.ConnectionState <> WebRTC.ConnectionState.NotConnected then
  WebRTC.setCandidate peer candidate
  model, Cmd.none
else
  { model with WaitingCandidates = candidate::model.WaitingCandidates }, Cmd.none
```

`ConnectionState` is set when `Offer` or `Answer` are received or when the `DataChannel` is open.

Handling `Answer`.

```fsharp
| WebRTC (WebRTC.Msg.Answer answer) ->
  WebRTC.setAnswer peer answer
  { model with ConnectionState = WebRTC.ConnectionState.Connecting }, Cmd.none
```

Going to `setAnswer` implementation.

```fsharp
let setAnswer (lifecycle: LifecycleModel) remote =
  try
    let desc = rtcSessionDescription remote
    internalLog (sprintf "setting answer: %A" desc)
    lifecycle.Peer.setRemoteDescription desc
    |> Promise.catch (sprintf "Failed to set remote description: %A" >> internalLog)
    |> Promise.start
  with e ->
    internalLog (sprintf "Error occured while adding remote description: %A" e)
```

As we could see, we only set `RemoteDescription` on a `PeerConnection` object here.

Right now, if everything succeeds, we should have a WebRTC connection ready. DataChannel is open between our peers. We could go to the code which is responsible for sending files. In UI, there should be a `textBox` that reacts on a file drop.

![send](https://mnie.github.com/img/2021_01_05_webrtc_ably/send.png)

Which is handled in the following way.

```fsharp
Field.div [
  Field.IsGrouped ] [
  Control.p [ Control.IsExpanded ] [
    div [   
      Class "card border-primary"
      Draggable true
      Style [ Height "100px" ]
      OnDragOver ( fun e ->
        e.preventDefault()
      )
      OnDrop ( fun e ->
        e.preventDefault()
        if e.dataTransfer.files.length > 0 then
          let file = e.dataTransfer.files.[0]
          { Name = file.name; Data = file.slice () }
          |> SendFile
          |> dispatch
      )
    ] []
  ]
]
```

`SendFile` message handling.

```fsharp
| SendFile file ->
  MessageTransfer.send peer (ChannelMessage.File file.Data)
  model, Cmd.none
```

Method `send` in `MessageTransfer`.

```fsharp
let send (lifecycle: LifecycleModel) (msg: ChannelMessage) =
  match msg, lifecycle.Channel with
  | File blob, Some channel ->
    blob
    |> U4.Case2
    |> channel.send
  | _, _ -> log "MessageTransfer" (sprintf "Unable to process: %A channel is: %A" msg lifecycle.Channel)
```

On a `receiver` side `onmessage` handling on `DataChannel` object looks as follows.

```fsharp
channel.onmessage <-
  fun e ->
    internalLog (sprintf "received msg in channel: %A" e.data)
    let blob = e.data :?> Blob
    msgHandler blob
```

As we could see, just for simplicity we assume that only javascript `blob` would be sent via `DataChannel`. We pass there also a `msgHandler` which is implemented in the following way.

```fsharp
let download (data: Blob) =
  let url = Browser.Url.URL.createObjectURL data
  let mutable anchor = document.createElement "a"
  anchor.hidden <- true
  anchor.setAttribute ("href", url)
  anchor.setAttribute ("download", "image.png")
  document.body.appendChild anchor |> ignore
  anchor.click ()
  document.body.removeChild anchor |> ignore
```

Its responsibility is to receive a `FileBlob` and immediately download it. We assume that the file would be always a `png` with the name `image`.

To sum up, thanks to the `Ably` platform I was able to implement `Signaling` in a simple way which would be worth consideration during the decision-making phase about “how to achieve Signaling in a most performant way” in my daily work. During some initial tests to compare it with standard HTTP request/response, it looks promising.

Another interesting thing is that a user can see how the messages/connections flow and work on the `Ably` dashboard. There is also an information about used quota.

![chart](https://mnie.github.com/img/2021_01_05_webrtc_ably/chart.png) ![quota](https://mnie.github.com/img/2021_01_05_webrtc_ably/quota.png)

Because there are no built-in signaling solutions Ably is for sure an almost `ready to go` alternative which we could use in our scenario. I hope I would be able to compare it with other ways to do signaling in the next articles.

[Repository](https://github.com/MNie/WebRTCSignaling)

I hope you enjoy it. Thanks for reading!