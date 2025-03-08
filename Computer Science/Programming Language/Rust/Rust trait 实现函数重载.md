#rust #typescript

函数重载和 Rust 的 trait 本质上都是 Ad-hoc polymorphism 的一种表现形式，两者表达能力基本相同，比如如下 C++ 中的函数重载

```cpp
int foo(int);
int foo(int, double);
```

在 Rust 中可以用 Trait + Tuple + Generic 模拟

```rust
trait FooImpl {
    fn foo_impl(self) -> i32;
}

impl FooImpl for (i32,) {
    fn foo_impl(self) -> i32 { .. }
}

impl FooImpl for (i32, f64) {
    fn foo_impl(self) -> i32 { .. }
}

fn foo<F: FooImpl>(f: F) -> i32 {
    f.foo_impl()
}
foo((1,));
foo((1,2.0));
```

只不过多了一层括号来表示这是个 Tuple。如果使用fn_trait 和 unboxed_closures，就能把这层括号给去了

```rust
struct foo;

impl FnOnce<(i32,)> for foo {
    type Output = i32;

    extern "rust-call" fn call_once(self, (a,): (i32,)) -> i32 {
        ...
    }
}

impl FnOnce<(i32,)> for foo {
    type Output = i32;
    
    extern "rust-call" fn call_once(self, (a, b): (i32, f64)) -> i32 {
        ...
    }
}

add(1);
add(1, 2.0);
```

C++ 中的函数重载中只能对参数类型重载，返回值类型是不能重载的。如果在上述的FooImpl trait 中把返回值类型作为 Generic 参数，就能实现对返回值类型的重载：

```rust
trait FooImpl<R> {
    fn foo_impl(self) -> R;
}

impl FooImpl<i32> for (i32,) {
    fn foo_impl(self) -> i32 { .. }
}

impl FooImpl<f64> for (i32,) {
    fn foo_impl(self) -> f64 { .. }
}

fn foo<R, F: FooImpl<R>>(f: F) -> R {
    f.foo_impl()
}

let i: i32 = foo((1,));
let j: f64 = foo((1,));
```

通过上述方式还可以实现更丧心病狂的东西：Variadic Function。因为 Rust 目前不支持 Variadic Generic，所以无法为任意长的 tuple 实现 trait，但我们可以通过把(A, B, C, ...) 转换为 (A, (B, (C, ...)))的形式来绕过这个限制，[tuple_list]() 这个库已经为我们做好了封装


例如，使用它我们可以实现一个接受任意多参数的concat
```rust
use tuple_list::{Tuple, TupleList};

trait ConcatImpl: TupleList {
    fn concat_impl(self) -> String;
}

impl ConcatImpl for () {
    fn concat_impl(self) -> String {
        "".into()
    }
}
impl<Head, Tail> ConcatImpl for (Head, Tail)
where
    Self: TupleList,
    Head: ToString,
    Tail: ConcatImpl,
{
    fn concat_impl(self) -> String {
        self.0.to_string() + &self.1.concat_impl()
    }
}

struct concat;
impl<'a, T> FnOnce<T> for concat
where
    T: tuple_list::Tuple + std::marker::Tuple,
    <T as Tuple>::TupleList: ConcatImpl,
{
    type Output = String;
    extern "rust-call" fn call_once(self, args: T) -> Self::Output {
        ConcatImpl::concat_impl(Tuple::into_tuple_list(args))
    }
}
```

然后

```rust
// (0.1 + 0.2 != 0.3) == true
concat("(", 0.1, " + 0.2 != ", 0.3, ") == ", true);
// 0.11false
concat(0.1, 1, false);
```

这就相当于cpp中的
```cpp
auto concat(auto &&fst, auto &&...res) {
    auto head = [&] {
        if constexpr (requires { std::string{fst}; })
            return std::string{fst};
        else
            return std::to_string(fst);
    }();
    if constexpr (sizeof...(res) == 0)
        return head;
    else
        return head + concat(std::forward<decltype(res)>(res)...);
}
```