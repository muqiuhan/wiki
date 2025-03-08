#rust #os #multi-thread 

用 epoll（只支持linux）实现一个最小功能的线程池异步运行时，效果为：
1. 实现一个 TcpStream，具体 api 仿照 std 里的 TcpStream，但是是异步版本
2. 实现一个在 `main` 上的 attribute proc macro，类似于 `tokio::main` 和 `tokio::test`，使得可以将 `main` 或单元测试变成异步函数，并且不破坏 rust analyzer 对于函数内容的类型标注、鼠标悬停提示等。

要求：不使用任何形式的锁，包含 `std` 给的锁、`parking_lot` 的锁或变相的自旋锁逻辑。

环境：
- 最新版 rustc，可使用所有的非 incomplete 的 unstable 功能。
- 可使用 std 和所有跟线程同步、异步基础设施无关的 crate。
- 代码中可有 unsafe，但不可存在 ub。
- 需要以下检查通过，并且没有任何错误/警告：
	`cargo miri test`
	`cargo clippy`
	`cargo fmt --check`