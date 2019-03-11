# Rustc Driver

这个[`rustc_driver`]本质上是`rustc`的`main()`函数。它充当了，要按正确顺序运行编译器各个阶段的粘合剂，管理诸如[`SourceMap`] （将 ast 节点映射到源代码），[`Session`] （一般构建上下文和错误消息）和[`TyCtxt`]
（“typing context”，允许您查询类型系统和其他很酷的东西）。这个`rustc_driver`箱还为外部用户，提供了在编译过程，特定时间运行代码的方法；允许第三方有效使用`rustc`的内部作为一个分析箱的库；或模拟编译程序进程（例如 RLS）。

对于那些使用`rustc`作为库，`run_compiler()`函数是编译器的主要入口。它的主要参数是一个命令行参数列表，和一个实现了`CompilerCalls`trait 的东东。一个`CompilerCalls`创建了整格`CompileController`，让它控制哪个传递近来的编译器运行，并在每个阶段结束时，附加上回调。

从`rustc_driver`从这个角度来看，编译器的主要阶段是：

1.  *解析输入：*初始的箱分析
2.  *配置和展开：*处理`#[cfg]`属性、名称解析和展开宏
3.  *运行分析过程：*在箱上运行 trait 解析、类型检查、区域检查和其他杂项分析。
4.  *转换为 llvm：*转换为 LLVM IR 的内存形式，并将其转换为可执行/对象文件。

之后，这个`CompileController`给予用户，介入编译时过程的能力

- 解析后
- AST 扩展后
- 降层为 HIR 后
- 分析后，以及
- 编译完成时

这个`CompileState`，各种各样的`state_after_*()`构造函数，可以检查并确定哪些信息位，用于哪个回调。

有关如何使用`rustc_driver`，检查一下[stupid-stats]指南(`@nrc`提供)（附为[附录 A][appendix a]）

> **警告：**从本质上讲，内部编译器 API 总是不稳定的。也就是说，如无必要，我们不会试图大搞破坏。

## 关于生命周期的

Rust 编译器是一个相当大的程序，包含大量的大数据结构（例如，AST、HIR 和类型系统），因此，大量依赖于 arena 和引用来减少不必要的内存使用。具体体现，就在人们可以插入编译器的方式上，更喜欢“push”式的 API（回调），而不是更可靠的“pull”式（想想`Iterator`trait）。

例如，[`CompileState`]是在每个阶段之后，传递给回调的状态，但本质上只是对编译器内部片段的一个可选引用(box 化)。之后，`CompilerCalls`trait 上的生命周期界限，则有助于确保编译器内部不能与编译器“分家”（例如，如果在编译器完成后，试图保持对 ast 的引用），同时，仍允许用户在`run_compiler()`函数完成之后，记录*一些*要用的状态，在。

线程-局部的存储和逗留 ，在编译器中被大量使用，以减少重复，同时也防止了，许多因大量存在的生命周期而引起的人体不适问题。这个`rustc::ty::tls`模块，用于访问这些线程-局部变量，尽管您很少需要接触它。

[`rustc_driver`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/
[`compilestate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/driver/struct.CompileState.html
[`session`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/session/struct.Session.html
[`tyctxt`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html
[`sourcemap`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceMap.html
[stupid-stats]: https://github.com/nrc/stupid-stats
[appendix a]: appendix/stupid-stats.html
