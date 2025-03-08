

## TL;DR

OCaml 的预处理器是指在编译时调用的程序，用于在实际编译之前修改程序。它们在处理文件包含、条件编译、样板生成或扩展语言的时候很有用。

一个例子:
```ocaml
Printf.printf "This program has been compiled by user: %s" [%get_env "USER"]
```

当使用预处理器编译代码时，它会将 `[%get_env "USER"]` 替换为 USER 环境变量的内容。例如，如果 USER 环境变量设置为 "JohnDoe"，则该行将变为：
```ocaml
Printf.printf "This program has been compiled by user: %s" "JohnDoe"
```

通过这种修改，预处理器在编译时将`[%get_env "USER"]`替换为`USER`环境变量的内容，这段代码在大多数系统上应该可以正常工作，而无需额外配置。编译时，预处理器会将`[%get_env "USER"]`替换为包含编译程序的用户的用户名的字符串。由于这是在编译时发生的，因此在运行时，`USER`变量的值不会产生影响，因为它仅用于编译程序中的信息目的。

## OCaml 中的预处理

某些语言内置支持预处理，意味着这类语言有一部分可以在编译时执行。例如，C语言的预处理器语法和语义是语言的一部分；Rust则有其宏系统。

OCaml 没有宏系统，所有预处理器都是独立的程序。尽管它不是语言的一部分，但 OCaml 平台官方支持用于编写此类预处理器的库。

具体来说，OCaml 支持两种类型的预处理器：一种在源级别工作（如C），另一种在 AST 上工作。后者称为"PPX"，即预处理器扩展（Pre-Processor eXtension）的缩写。

虽然这两种预处理都有其使用场景，但在 OCaml 中，建议尽可能使用 PPX，原因有几个：

- 与 Merlin 和 Dune 集成得很好，不会干扰编辑器中的错误报告和 Merlin 的“跳转到定义”功能。
- 性能好，易于组合。
- 与 OCaml 特性相辅相成。


## Refs.
- [Preprocessors and PPXs](https://ocaml.org/docs/metaprogramming#metaprogramming)
- [A Tutorial to OCaml -ppx Language Extensions](https://www.victor.darvariu.me/jekyll/update/2018/06/19/ppx-tutorial.html)
- [ppxlib: quick_intro](https://ocaml-ppx.github.io/ppxlib/ppxlib/quick_intro.html)
- [An introduction to OCaml ppx ecosystem](https://tarides.com/blog/2019-05-09-an-introduction-to-ocaml-ppx-ecosystem/)