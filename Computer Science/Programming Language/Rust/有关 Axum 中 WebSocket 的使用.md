#rust #web

## 创建 `axum` 项目

创建一个项目，并将 `axum` 添加到依赖中：

```sh
cargo new axum-ws-test
cd axum-ws-test
cargo add tokio -F full
cargo add serde_json
cargo add axum -F ws
cargo add rand
```

然后用自己喜欢的编辑器/IDE 打开整个项目，找到 `Cargo.toml`，可以看到 `Cargo.toml` 如下：

```toml
[dependencies]
axum = { version = "0.6.20", features = ["ws"] }
rand = "0.8.5"
serde_json = "1.0.107"
tokio = { version = "1.33.0", features = ["full"] }
```

> 以上依赖版本为本文编写时的最新稳定版，需要注意和自己的版本区别，`axum` 的功能基本都有解释，可以查看 [`axum` 文档](https://docs.rs/axum/latest/axum/#feature-flags)

### `axum` 的程序基本结构

从官方文档可以看到，一个 `axum` 程序，包含了程序入口、路由、路由服务和 `axum` 服务端（即 `axum::Server`），我们先将其基本结构写入到 `main.rs` 的文件中（代码来自官方文档）：

```rust
use axum::{routing::get, Router};

// 主函数入口
#[tokio::main]
async fn main() {
    // 路由
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    // axum 的 Server
    axum::Server::bind(&"0.0.0.0:8081".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

运行程序后，打开 [localhost:8081](localhost:8081)，应该可以看见网页上有 `Hello, World!`。

## 创建前端项目

前端使用 `Vue`，建议选择另一个文件夹来创建前端项目。输入下面的指令来创建前端项目，项目名称命名为 `axum-test-front`：

```sh
npm create vue@latest
cd axum-test-front
npm install
npm run dev
```

## 直接使用 Http 请求

我们模拟的情况试试，前端每次点击按钮都会获取后端的一个随机数。一开始我们先不使用 WebSocket，来测试一下效果。

### 后端代码

添加一个函数，用于前端获取随机数：

```rust
use axum::{response::Json, routing::get, Router};
use rand::Rng;
use serde_json::{json, Value};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        // add here
        .route("/random", get(get_rand));

    // Server
}
// handler
async fn get_rand() -> Json<Value> {
    let mut rng = rand::thread_rng();
    Json(json! ({"num": rng.gen_range(1..=100)}))
}
```

可以在浏览器中输入 [localhost:8081/random](localhost:8081/random)来测试，每个刷新应该都可以得到一个新的 `num` 值。

### 前端代码

前端在 `App.vue` 中添加一个按钮和一个用于显示获取到的随机数的节点，代码如下：

```html
<script>
  export default {
    data() {
      return {
        num: null,
      };
    },
    methods: {
      get_random() {
        // 从后端的对应地址获取随机数
        fetch("http://localhost:8081/random", {
          mode: "cors",
          headers: {
            accpet: "application/json",
          },
        })
          .then((response) => response.json())
          .then((data) => {
            this.num = data.num;
          });
      },
    },
  };
</script>

<template>
  <main>
    <button @click="get_random()">click to get num</button>
    <div>num is {{ num }}</div>
  </main>
</template>
```

执行的时候会发现无法从后端拿到数据，这是因为后端没有配置跨域请求。

### 为后端配置跨域

首先需要添加一个依赖，输入下面的指令添加：

```sh
cargo add tower-http -F cors
```

然后在创建路由之前，先新建一个跨域的许可：

```rust
use axum::{http::HeaderValue, response::Json, routing::get, Router};
use rand::Rng;
use serde_json::{json, Value};
use tower_http::cors::{Any, CorsLayer};

#[tokio::main]
async fn main() {
    // 跨域配置
    let cors = CorsLayer::new()
        .allow_methods(Any)
        .allow_headers(Any)
        .allow_origin("http://localhost:5173".parse::<HeaderValue>().unwrap());

    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        // 为路由方法处理添加跨域许可
        .route("/random", get(get_rand).layer(cors));

    // Server
}
```

这时再运行后端和前端，打开前端的网页，点击按钮应该可以每次获取到不同的数字。打开开发者控制台，并选择网络（没有的话点加号或者 `》` 可以找到），再多次点击按钮，可以看到每次点击按钮都发送了一次 Http 请求。

> 如果前端开发者控制台报 `Uncaught (in promise) ReferenceError: num is not defined` 这种错误，应该是在给 Vue 中 `data` 里的字段赋值的时候没有加 `this` 关键字，把 `num` 改为 `this.num` 即可

# 改为 WebSocket 传输随机数

## 后端代码修改

添加一个新的函数，函数名为 `handle_random`，再添加一个函数名为 `handle_random_socket`：

```rust
async fn handle_random(ws_upgrade: WebSocketUpgrade) -> Response {
    ws_upgrade.on_upgrade(handle_random_socket)
}

async fn handle_random_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg {
            msg
        } else {
            println!("Web Socket Closed");
            return;
        };

    }
}
```

在 `handle_random` 中，调用 `ws_upgrade` 的 `on_upgrade` 函数可以建立 Web Socket 连接。`handle_random_socket` 就是用于处理连接时的函数，此处先使用循环来接收来自连接另一端的消息，如果接收发生错误或者无法接收到，则视为连接关闭。`WebSocket::recv()` 函数在连接关闭后，才会返回 `None`。

设想建立连接后，前端发送一个 `get` 字符串，后端收到这个字符串，如果收到的确实是 `get`，则返回给前端一个随机数。那么要做的事情就很简单了，首先要匹配发送过来的消息是否是字符串且内容是否为 `get`。

```rust
async fn handle_random_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg {...};
        // 匹配字符串
        if let Message::Text(text) = msg {
            if text.eq("get") {
                todo!()
            }
        }
    }
}
```

在匹配成功后，将生成一个随机数，并返回给前端。这里会用到 `WebSocket` 的 `Send` 函数来返回响应，获取随机数可以用到之前写的函数。前端对数据的宽容性较大，所以可以考虑直接返回 JSON 格式的文本：

```rust
async fn handle_random_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg {...};

        if let Message::Text(text) = msg {
            if text.eq("get") {
                if socket
                    // 返回随机数
                    .send(Message::Text(get_rand().await.to_string()))
                    .await
                    .is_err()
                {
                    // 如果出错了就关闭连接
                    println!("Web Socket Closed");
                    return;
                }
            }
        }
    }
    println!("Web Socket Closed");
}
```

编写完成，最后将 `handle_random` 添加到路由中。

```rust
#[tokio::main]
async fn main() {
    // 跨域配置
    // ...
    // 路由
    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route("/random", get(get_rand).layer(cors));
        .route("/ws/random", get(handle_random));

    // axum 的 Server
    // ...
}
```

## 前端代码修改

在组件挂载时，会尝试创建一个 `WebSocket` 的连接，并且绑定一个函数，用于在收到消息后，设置 `num` 的值：

```js
<script>
export default {
  data() {
    return {
      num: null,
      ws: null,
    };
  },
  methods: {
    build_connect() {
      // 防止子域的 this 与 vue 的 this 冲突
      var that = this;
      that.ws = new WebSocket("ws://localhost:8081/ws/random");
      // 收到消息后，设置 num 的值
      that.ws.addEventListener("message", function (event) {
        that.num = JSON.parse(event.data).num;
      });
    },
  },
  mounted() {
    this.build_connect();
    window.onclose = () => {
      this.ws.close();
    };
  },
};
```

之后的每次点击都会变为通过连接来向后端发送消息。按照逻辑修改后的代码如下：

```js
get_random() {
  // 通过连接向后端发送信息
  this.ws.send("get");
},
```

## 运行

首先启动后端：

```sh
cargo run
```

然后启动前端：

```sh
npm run dev
```

打开前端页面，点击按钮就可以看到每次都能够从后端获取到不同的随机数的效果了。

# 参考

1. [axum](https://docs.rs/axum)
    - [axum::extract::ws - Rust](https://docs.rs/axum/latest/axum/extract/ws/index.html)
2. [vue](https://cn.vuejs.org/)
3. [WebSocket - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)