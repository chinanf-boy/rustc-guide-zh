# 调试编译器

[debugging]: #debugging

本章包含一些调试编译器的技巧。无论您在做什么，希望这些技巧能帮到你。其他一些章节都有对编译器某部分的建议（例如[查询调试和测试章节](./incrcomp-debugging.zh.html)或者[LLVM 调试章节](./codegen/debugging.zh.md)）。

## `-Z`标志

编译器有很多`-Z`标志。这些是不稳定的标志，只在夜间启用。其中许多对调试很有用。要获得`-Z`标志的完整列表，使用`-Z help`。

一个有用的标志是`-Z verbose`，通常可以打印，关于调试的更多信息。

## 获得回溯

[getting-a-backtrace]: #getting-a-backtrace

当你有 ICE（编译器中的恐慌）时，你可以设置`RUST_BACKTRACE=1`得到`panic!`的堆栈跟踪，就像在正常的 Rust 程序中一样。IIRC 回溯在 Mac 和 MinGW 上，是**不工作**的，抱歉。如果你遇到麻烦或者回溯充满了`unknown`，您也许要想办法，使用 Linux 或 Windows 的 MSVC。

在默认配置中，您没有启用代码行编号，因此回溯如下所示：

```text
stack backtrace:
   0: std::sys::imp::backtrace::tracing::imp::unwind_backtrace
   1: std::sys_common::backtrace::_print
   2: std::panicking::default_hook::{{closure}}
   3: std::panicking::default_hook
   4: std::panicking::rust_panic_with_hook
   5: std::panicking::begin_panic
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
  32: rustc_typeck::check_crate
  33: <std::thread::local::LocalKey<T>>::with
  34: <std::thread::local::LocalKey<T>>::with
  35: rustc::ty::context::TyCtxt::create_and_enter
  36: rustc_driver::driver::compile_input
  37: rustc_driver::run_compiler
```

如果需要堆栈跟踪的行号，则可以启用`debuginfo-lines=true`，要么在 config.toml 中变为 `debuginfo=true`，并重建编译器。然后回溯将如下所示：

```text
stack backtrace:
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
             at /home/user/rust/src/librustc_typeck/check/cast.rs:110
   7: rustc_typeck::check::cast::CastCheck::check
             at /home/user/rust/src/librustc_typeck/check/cast.rs:572
             at /home/user/rust/src/librustc_typeck/check/cast.rs:460
             at /home/user/rust/src/librustc_typeck/check/cast.rs:370
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
  33: rustc_driver::driver::compile_input
             at /home/user/rust/src/librustc_driver/driver.rs:1010
             at /home/user/rust/src/librustc_driver/driver.rs:212
  34: rustc_driver::run_compiler
             at /home/user/rust/src/librustc_driver/lib.rs:253
```

## 获取错误的回溯

[getting-a-backtrace-for-errors]: #getting-a-backtrace-for-errors

如果要获得回溯到编译器发出错误消息的位置，可以传递`-Z treat-err-as-bug`，这让编译器对它看到的第一个错误，恐慌回溯。

这`delay_span_bug`的调用，在调试时也有帮助-第一个`delay_span_bug`的调用会引起恐慌，给你一个有用的回溯。

例如：

```bash
$ cat error.rs
fn main() {
    1 + ();
}
```

```bash
$ ./build/x86_64-unknown-linux-gnu/stage1/bin/rustc error.rs
error[E0277]: the trait bound `{integer}: std::ops::Add<()>` is not satisfied
 --> error.rs:2:7
  |
2 |     1 + ();
  |       ^ no implementation for `{integer} + ()`
  |
  = help: the trait `std::ops::Add<()>` is not implemented for `{integer}`

error: aborting due to previous error

$ # 现在， 如何知道错误来自哪里?
$ RUST_BACKTRACE=1 \
    ./build/x86_64-unknown-linux-gnu/stage1/bin/rustc \
    error.rs \
    -Z treat-err-as-bug
error[E0277]: the trait bound `{integer}: std::ops::Add<()>` is not satisfied
 --> error.rs:2:7
  |
2 |     1 + ();
  |       ^ no implementation for `{integer} + ()`
  |
  = help: the trait `std::ops::Add<()>` is not implemented for `{integer}`

error: internal compiler error: unexpected panic

note: the compiler unexpectedly panicked. this is a bug.

note: we would appreciate a bug report: https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md#bug-reports

note: rustc 1.24.0-dev running on x86_64-unknown-linux-gnu

note: run with `RUST_BACKTRACE=1` for a backtrace

thread 'rustc' panicked at 'encountered error with `-Z treat_err_as_bug',
/home/user/rust/src/librustc_errors/lib.rs:411:12
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose
backtrace.
stack backtrace:
  (~~~ IRRELEVANT PART OF BACKTRACE REMOVED BY ME ~~~)
   7: rustc::traits::error_reporting::<impl rustc::infer::InferCtxt<'a, 'gcx,
             'tcx>>::report_selection_error
             at /home/user/rust/src/librustc/traits/error_reporting.rs:823
   8: rustc::traits::error_reporting::<impl rustc::infer::InferCtxt<'a, 'gcx,
             'tcx>>::report_fulfillment_errors
             at /home/user/rust/src/librustc/traits/error_reporting.rs:160
             at /home/user/rust/src/librustc/traits/error_reporting.rs:112
   9: rustc_typeck::check::FnCtxt::select_obligations_where_possible
             at /home/user/rust/src/librustc_typeck/check/mod.rs:2192
  (~~~ IRRELEVANT PART OF BACKTRACE REMOVED BY ME ~~~)
  36: rustc_driver::run_compiler
             at /home/user/rust/src/librustc_driver/lib.rs:253
$ # 好啊，现在我有了错误的回溯
```

## 获取日志输出

[getting-logging-output]: #getting-logging-output

这些箱子用于编译器中的日志记录：

- [log]
- [env-logger]：检查链接以查看完整`RUST_LOG`语法

[log]: https://docs.rs/log/0.4.6/log/index.html
[env-logger]: https://docs.rs/env_logger/0.4.3/env_logger/

编译器有很多`debug!`调用，在多个位置点打印出日志信息。如果不能完全找到一个 bug，那么对缩小 bug 的范围非常有用，或者仅仅是为了让你熟悉编译器为什么要做这件事情。

要查看日志，需要设置`RUST_LOG`环境变量，其有关日志筛选，例如，要获取特定模块的日志，可以以如下方式运行编译器`RUST_LOG=module::path rustc my-file.rs`。全部的`debug!`稍后会输出在标准错误中。

请注意，除非，使用非常严谨的筛选器，否则日志会有*许多*输出-所以通常最好把标准错误传输到文件，之后使用文本编辑器查看日志输出。

所以把它放在一起。

```bash
# 这会将`librustc/traits`的所有调试，调用的输出放入  终端的标准错误通道
# , 堆满你的终端屏幕
$ RUST_LOG=rustc::traits rustc +local my-file.rs

# 这会将 `librustc/traits`的所有调试，调用的输出放入
# `traits-log`, 稍后用文件查看器查看.
$ RUST_LOG=rustc::traits rustc +local my-file.rs 2>traits-log

# 不推荐。这将显示所有'debug!'的输出。
# 在Rust编译器中，它们有很多，所以
# 很难找到任何东西。
$ RUST_LOG=debug rustc +local my-file.rs 2>all-log

# 这将“rustc_trans”调用，显示所有信息的输出。
#
# 这`trans_instance`里面有个`info!`表达式语句，会
# 帮每个函数转成易懂。对查找哪个出发了一个LLVM断言，很有用。
# 这就是一个`info!`，而不是一个`debug!`，最终落户官方编译器
$ RUST_LOG=rustc_trans=info rustc +local my-file.rs
```

### 如何保存或移除`debug!`和从生成的二进制文件调用`trace!`

`error!`，`warn!`和`info!`的调用包含在编译器的每个版本中；`debug!`和`trace!`的调用只有在以下情况下才包括在程序中：在 config.toml 中，用`debug-assertions=yes`启用（默认情况下关闭），因此如果您没有看到`DEBUG`日志，尤其是当使用`RUST_LOG=rustc rustc some.rs`，却只看到`INFO`日志，要记得在 config.toml 中，把`debug-assertions=yes`打开。

我还在想啊，在某些情况下，仅仅设置它，是不会触发重建，因此如果您更改了设置，并且已有个构建，那么您可以用`x.py clean`，强迫重建。

### 日志礼仪与习俗

因为`debug!`的调用，默认情况下会被删除，在大多数情况下，不要担心向添加“不必要”的`debug!`调用，就把它们留在您提交的代码中 —— 它们不会降低我们发布版本的性能，如果日志能帮助您消除了一个 bug，也可能会帮助其他人解决另一个 bug。

一个通俗的惯例是：在一个`foo`函数的*开始*，使用`debug!("foo(...)")`；和在其*在函数内部*，使用`debug!("foo: ...")`。另一个惯例是使用`{:?}`，作为调试日志的格式说明符。

一件**需要关心**的事情是，日志中的**昂贵的**操作。

如果在`rustc::foo`模块中，你有一个声明

```Rust
debug!("{:?}", random_operation(tcx));
```

如果有人用`RUST_LOG=rustc::bar`，运行`rustc`调试，然后`random_operation()`会调用。

这意味着，你不应该把任何太贵或可能崩溃的东西放进去 —— 这会使任何想把日志记录使用在自己模块的人，恼火！！除非有人试图使用日志查找*另一个*bug。

## 格式化 graphviz 输出（.dot 文件）

[formatting-graphviz-output]: #formatting-graphviz-output

用于调试特定功能的一些编译器选项，没错，图形化 —— 例如`#[rustc_mir(borrowck_graphviz_postflow="suffix.dot")]`属性，能转储各种借用检查器的数据流图。

这些都会产生`.dot`文件。要查看这些文件，请安装 graphviz（例如`apt-get install graphviz`）然后运行以下命令：

```bash
$ dot -T pdf maybe_init_suffix.dot > maybe_init_suffix.pdf
$ firefox maybe_init_suffix.pdf # 或其他PDF阅读器
```

## 缩小（Bisecting）性能退步

这个[cargo-bisect-rustc][bisect]工具可以作为一种快速而简单的方法来，精确找出哪个 PR 导致`rustc`行为变化。它会自动下载对应 PR 的`rustc`源代码，并根据您提供的项目对它们进行测试，直到找到退步为止。然后您可以查看 PR 以了解，*为什么*它被改变了的更多内容。见[本教程][bisect-tutorial]关于如何使用它。

[bisect]: https://github.com/rust-lang-nursery/cargo-bisect-rustc
[bisect-tutorial]: https://github.com/rust-lang-nursery/cargo-bisect-rustc/blob/master/TUTORIAL.md

## 从 Rust 的 CI 下载工件

这个[rustup-toolchain-install-master][rtim] 工具(来自 kennytm)可以用来下载， 由 Rust 的 CI(特定 SHA1)生成的工件 —— 这基本上，相当于一些成功 pr 的结果 —— 然后将它们设置为本地使用。该工具能用在`@bors try`。当您希望在不进行构建的情况下，检查生成的 PR 构建工件时，这很有帮助。

[rtim]: https://github.com/kennytm/rustup-toolchain-install-master
