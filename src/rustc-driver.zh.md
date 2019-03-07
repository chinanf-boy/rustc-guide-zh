# 鲁斯特司机

这个[`rustc_driver`]本质上是`rustc`的`main()`功能。它充当了按正确顺序运行编译器各个阶段的粘合剂，管理诸如[`SourceMap`] （将ast节点映射到源代码）。[`Session`] （常规生成上下文和错误消息）和[`TyCtxt`]
（“输入上下文”，允许您查询类型系统和其他很酷的东西）。这个`rustc_driver`板条箱还为外部用户提供了在编译过程中的特定时间运行代码的方法，允许第三方有效地使用`rustc`作为分析板条箱或模拟编译程序进程（例如RLS）的库的内部。

对于那些使用`rustc`作为图书馆，`run_compiler()`函数是编译器的主要入口点。它的主要参数是命令行参数列表和对实现`CompilerCalls`特质。一`CompilerCalls`创建整体`CompileController`，让它控制运行哪个编译器传递，并在每个阶段结束时附加要激发的回调。

从`rustc_driver`从这个角度来看，编译器的主要阶段是：

1.  *解析输入：*初始板条箱分析
2.  *配置和展开：*决心`#[cfg]`属性、名称解析和展开宏
3.  *运行分析过程：*在板条箱上运行特性解析、类型检查、区域检查和其他杂项分析。
4.  *转换为llvm：*转换为llvm-ir的内存形式，并将其转换为可执行/对象文件

这个`CompileController`然后让用户能够检查正在进行的编译过程

-   解析后
-   AST扩展后
-   HIR降低后
-   分析后，以及
-   编译完成时

这个`CompileState`各种各样的`state_after_*()`可以检查构造函数以确定哪些信息位可用于哪个回调。

有关如何使用`rustc_driver`检查一下[愚蠢的统计-]指南`@nrc`（附为[附录A]）

> **警告：**从本质上讲，内部编译器API总是不稳定的。也就是说，我们不会试图不必要地破坏事物。

## 关于生命周期的笔记

Rust编译器是一个相当大的程序，包含大量的大数据结构（例如，AST、HIR和类型系统），因此，大量依赖于Arena和引用来减少不必要的内存使用。这体现在人们可以插入编译器的方式上，更喜欢“推”式API（回调），而不是更可靠的“拉”式（想想`Iterator`特质）。

例如[`CompileState`]，在每个阶段之后传递给回调的状态，本质上只是对编译器内部片段的一个可选引用框。生命的界限`CompilerCalls`然后，特征有助于确保编译器内部不会“转义”编译器（例如，如果在编译器完成后试图保持对ast的引用），同时仍允许用户记录*一些*之后使用的状态`run_compiler()`功能完成。

线程本地存储和interning在编译器中被大量使用，以减少重复，同时也防止了许多由于普遍存在的生命周期而引起的人体工程学问题。这个`rustc::ty::tls`模块用于访问这些线程局部变量，尽管您很少需要接触它。

[`rustc_driver`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/

[`compilestate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/driver/struct.CompileState.html

[`session`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/session/struct.Session.html

[`tyctxt`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html

[`sourcemap`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceMap.html

[stupid-stats]: https://github.com/nrc/stupid-stats

[appendix a]: appendix/stupid-stats.html
