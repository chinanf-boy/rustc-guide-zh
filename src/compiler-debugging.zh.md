# 调试编译器

[debugging]: #debugging

本章包含一些调试编译器的技巧。无论您在做什么，这些技巧都旨在发挥作用。其他一些章节对编译器的特定部分有建议（例如[查询调试和测试章节](./incrcomp-debugging.html)或者[LLVM调试章节](./codegen/debugging.md)）。

## `-Z`旗

编译器有很多`-Z`标志。这些是不稳定的标志，只在夜间启用。其中许多对调试很有用。要获得完整列表`-Z`旗帜，使用`-Z help`。

一个有用的标志是`-Z verbose`，通常可以打印更多可用于调试的信息。

## 获得回溯

[getting-a-backtrace]: #getting-a-backtrace

当你有ICE（编译器中的恐慌）时，你可以设置`RUST_BACKTRACE=1`得到的堆栈跟踪`panic!`像在正常的Rust程序中一样。IIRC回溯**不工作**在Mac和MinGW上，抱歉。如果你遇到麻烦或者回溯充满了`unknown`，您可能想找到一些在Windows上使用Linux或MSVC的方法。

在默认配置中，您没有启用行号，因此回溯如下所示：

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

如果需要堆栈跟踪的行号，则可以启用`debuginfo-lines=true`要么`debuginfo=true`在config.toml中并重建编译器。然后回溯将如下所示：

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

如果要获得回溯到编译器发出错误消息的位置，可以传递`-Z treat-err-as-bug`，这将使编译器对它看到的第一个错误感到恐慌。

这在调试时也有帮助`delay_span_bug`打电话-会先打`delay_span_bug`呼叫恐慌，这将给你一个有用的回溯。

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

$ # Now, where does the error above come from?
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
$ # Cool, now I have a backtrace for the error
```

## 获取日志输出

[getting-logging-output]: #getting-logging-output

这些板条箱用于编译器中的日志记录：

-   [日志]
-   [环境记录器-]：检查链接以查看完整`RUST_LOG`语法

[log]: https://docs.rs/log/0.4.6/log/index.html

[env-logger]: https://docs.rs/env_logger/0.4.3/env_logger/

编译器有很多`debug!`调用，它在多个点打印出日志信息。如果不能完全找到一个bug，或者仅仅是为了确定编译器为什么要做一件特定的事情，那么这些功能对于缩小bug的范围非常有用。

要查看日志，需要设置`RUST_LOG`日志筛选器的环境变量，例如，要获取特定模块的日志，可以以如下方式运行编译器`RUST_LOG=module::path rustc my-file.rs`.全部`debug!`然后输出将出现在标准错误中。

请注意，除非使用非常严格的筛选器，否则记录器将发出*许多*输出-所以通常最好将标准错误传输到文件，并使用文本编辑器查看日志输出。

所以把它放在一起。

```bash
# This puts the output of all debug calls in `librustc/traits` into
# standard error, which might fill your console backscroll.
$ RUST_LOG=rustc::traits rustc +local my-file.rs

# This puts the output of all debug calls in `librustc/traits` in
# `traits-log`, so you can then see it with a text editor.
$ RUST_LOG=rustc::traits rustc +local my-file.rs 2>traits-log

# Not recommended. This will show the output of all `debug!` calls
# in the Rust compiler, and there are a *lot* of them, so it will be
# hard to find anything.
$ RUST_LOG=debug rustc +local my-file.rs 2>all-log

# This will show the output of all `info!` calls in `rustc_trans`.
#
# There's an `info!` statement in `trans_instance` that outputs
# every function that is translated. This is useful to find out
# which function triggers an LLVM assertion, and this is an `info!`
# log rather than a `debug!` log so it will work on the official
# compilers.
$ RUST_LOG=rustc_trans=info rustc +local my-file.rs
```

### 如何保存或移除`debug!`和`trace!`从生成的二进制文件调用

当呼叫`error!`，请`warn!`和`info!`包含在编译器的每个版本中，调用`debug!`和`trace!`只有在以下情况下才包括在程序中`debug-assertions=yes`在config.toml中打开（默认情况下关闭），因此如果您没有看到`DEBUG`日志，尤其是当使用`RUST_LOG=rustc rustc some.rs`只看到`INFO`日志，确保`debug-assertions=yes`在config.toml中打开。

我还认为，在某些情况下，仅仅设置它不会触发重建，因此如果您更改了它，并且已经构建了编译器，那么您可能需要调用`x.py clean`强迫一个。

### 伐木礼仪与习俗

因为打电话给`debug!`默认情况下会被删除，在大多数情况下，不要担心向添加“不必要”调用`debug!`把它们留在您提交的代码中——它们不会降低我们发布的代码的性能，如果它们帮助您消除了一个bug，它们可能会帮助其他人解决另一个bug。

一个不严格遵循的惯例是`debug!("foo(...)")`在*开始*函数的`foo`和`debug!("foo: ...")` *在内部*功能。另一个松散遵循的约定是使用`{:?}`调试日志的格式说明符。

一件事**仔细的**IS的**昂贵的**日志中的操作。

如果在模块中`rustc::foo`你有一个声明

```Rust
debug!("{:?}", random_operation(tcx));
```

如果有人运行调试`rustc`具有`RUST_LOG=rustc::bar`，然后`random_operation()`将运行。

这意味着你不应该把任何太贵或可能崩溃的东西放在那里-这会使任何想为自己的模块使用日志记录的人恼火。除非有人试图使用日志查找*另一个*缺陷。

## 格式化graphviz输出（.dot文件）

[formatting-graphviz-output]: #formatting-graphviz-output

用于调试特定功能的一些编译器选项生成图形化图形-例如`#[rustc_mir(borrowck_graphviz_postflow="suffix.dot")]`属性转储各种借用检查器数据流图。

这些都会产生`.dot`文件夹。要查看这些文件，请安装graphviz（例如`apt-get install graphviz`）然后运行以下命令：

```bash
$ dot -T pdf maybe_init_suffix.dot > maybe_init_suffix.pdf
$ firefox maybe_init_suffix.pdf # Or your favorite pdf viewer
```

## 缩小（平分）回归

这个[货物平分生锈][bisect]工具可以作为一种快速而简单的方法来精确地找出导致`rustc`行为。它会自动下载`rustc`pr工件，并根据您提供的项目对它们进行测试，直到找到回归为止。然后您可以查看公关以了解更多内容*为什么*它被改变了。见[本教程][bisect-tutorial]关于如何使用它。

[bisect]: https://github.com/rust-lang-nursery/cargo-bisect-rustc

[bisect-tutorial]: https://github.com/rust-lang-nursery/cargo-bisect-rustc/blob/master/TUTORIAL.md

## 从Rust的CI下载工件

这个[生锈工具链安装主][rtim]KennyTM的工具可以用来下载Rust的CI为特定的sha1生成的工件——这基本上相当于一些pr的成功登陆——然后将它们设置为本地使用。这也适用于由`@bors
try`. 当您希望在不进行构建的情况下检查生成的PR构建时，这很有帮助。

[rtim]: https://github.com/kennytm/rustup-toolchain-install-master
