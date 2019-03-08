# `compiletest`

## 介绍

`compiletest`是 Rust 测试套件的主要测试工具。它允许测试作者组织大量测试（Rust 编译器有数千个），高效的测试执行（支持并行执行），并允许测试作者配置个人和测试组的行为和预期结果。

`compiletest`测试可以检查测试代码是否成功，失败或在某些情况下甚至是编译失败。测试通常被组织为 Rust 源文件，在测试代码之前和/或之内的注释中带有注释，用于指导`compiletest`如果或如何运行测试，期望什么行为，等等。如果您不熟悉编译器测试框架，请参阅[本章](./tests/intro.html)了解更多背景资料

测试本身通常（但不总是）组织成“套房” - 例如，`run-pass`，一个表示应该成功的测试的文件夹，`run-fail`，一个包含应该成功编译的测试的文件夹，但是返回失败（非零状态），`compile-fail`，一个包含应该无法编译的测试的文件夹，还有更多。各种套房的定义[SRC /工具/ compiletest / src 目录/ common.rs][common]在里面`pub struct Config`宣言。有关不同套件编译器测试的详细介绍以及有关它们的详细信息，请参阅[添加新测试](./tests/adding.html)。

## 添加新的测试文件

简而言之，只需在适当的位置创建新测试[SRC /测试][test]。不需要注册测试文件`compiletest`将扫描[SRC /测试][test]子文件夹递归，并将执行它作为测试找到的任何 Rust 源文件。看到[`Adding new tests`](./tests/adding.html)有关如何添加新测试的完整指南。

## 标头命令

源文件注释出现在源文件顶部附近的注释中*之前*任何测试代码都称为头命令。这些命令可以指示`compiletest`忽略这个测试，设定对预期是否会成功编译的期望，或预期测试的返回代码是什么。标题命令（及其内联对应项，错误信息命令）将更全面地描述[这里](./tests/adding.html#header-commands-configuring-rustc)。

### 添加新的头命令

标题命令在。中定义`TestProps`结构[SRC /工具/ compiletest / src 目录/ header.rs][header]。在较高的层次上，这里定义了许多测试属性，所有测试属性都设置为默认值`TestProp`结构的`impl`块。任何测试都可以通过将有问题的属性指定为标题命令作为注释来覆盖此默认值（`//`）在测试源文件中，在任何源代码之前。

#### 使用 header 命令

这是一个例子，指定`must-compile-successfully`header 命令，不带参数，后跟`failure-status`header 命令，它接受一个参数（在这种情况下，值为 1）。`failure-status`指示`compiletest`期望失败状态为 1（而不是在撰写本文时当前的 Rust 默认值为 101）。header 命令和参数列表（如果存在）通常用冒号分隔：

```rust,ignore
// must-compile-successfully
// failure-status: 1

#![feature(termination_trait)]

use std::io::{Error, ErrorKind};

fn main() -> Result<(), Box<Error>> {
    Err(Box::new(Error::new(ErrorKind::Other, "returned Box<Error> from main()")))
}
```

#### 添加新的标头命令属性

如果需要在逐个测试的基础上定义一些测试属性或行为，则可以添加新的头命令。头命令属性在运行时用作头命令的后备存储（保存命令的当前值）。

要添加新的标头命令属性：1。查找声明`pub struct TestProps`SRC /工具/ compiletest / src 目录/ header.rs[并将新的公共属性添加到声明的末尾。][header]2.寻找`impl TestProps`紧跟结构声明后的实现块，并将新属性初始化为其默认值。

#### 添加新的头命令解析器

什么时候`compiletest`遇到一个测试文件，它通过调用中定义的每个解析器一次解析一行文件`Config`struct 的实现块，也在[SRC /工具/ compiletest / src 目录/ header.rs][header]（注意`Config`struct 的声明块可以在[SRC /工具/ compiletest / src 目录/ common.rs][common]。`TestProps`的`load_from()`方法将尝试将当前文本行传递给每个解析器，然后通常会检查该行是否以特定注释开头（`//`）header 命令如`// must-compile-successfully`要么`// failure-status`。注释标记后面的空格是可选的。

解析器将仅通过在测试文件中将其指定为头命令或通过在测试文件中指定参数值来覆盖给定的头命令属性的默认值，具体取决于头命令。

解析器定义于`impl Config`通常被命名`parse_<header_command>`（注意烤肉串`<header-command>`转变为蛇案`<header_command>`）。`impl Config`还定义了几个“低级”解析器，这使得解析常见模式（如简单存在与否）变得简单（`parse_name_directive()`），header-command：参数（s）（`parse_name_value_directive()`），可选解析只有特定的`cfg`属性已定义（`has_cfg_prefix()`） 还有很多。低级解析器位于接近末尾`impl Config`块;请务必仔细查看它们及其上面的相关解析器，看看它们是如何被用来避免不必要地编写额外的解析代码的。

作为一个具体的例子，这里是实现的`parse_failure_status()`解析器，在[SRC /工具/ compiletest / src 目录/ header.rs][header]：

```diff
@@ -232,6 +232,7 @@ pub struct TestProps {
     // customized normalization rules
     pub normalize_stdout: Vec<(String, String)>,
     pub normalize_stderr: Vec<(String, String)>,
+    pub failure_status: i32,
 }

 impl TestProps {
@@ -260,6 +261,7 @@ impl TestProps {
             run_pass: false,
             normalize_stdout: vec![],
             normalize_stderr: vec![],
+            failure_status: 101,
         }
     }

@@ -383,6 +385,10 @@ impl TestProps {
             if let Some(rule) = config.parse_custom_normalization(ln, "normalize-stderr") {
                 self.normalize_stderr.push(rule);
             }
+
+            if let Some(code) = config.parse_failure_status(ln) {
+                self.failure_status = code;
+            }
         });

         for key in &["RUST_TEST_NOCAPTURE", "RUST_TEST_THREADS"] {
@@ -488,6 +494,13 @@ impl Config {
         self.parse_name_directive(line, "pretty-compare-only")
     }

+    fn parse_failure_status(&self, line: &str) -> Option<i32> {
+        match self.parse_name_value_directive(line, "failure-status") {
+            Some(code) => code.trim().parse::<i32>().ok(),
+            _ => None,
+        }
+    }
```

## 实现行为改变

当测试调用特定的头命令时，预计某些行为将因此而改变。显然，什么行为将取决于 header 命令的用途。如果是`failure-status`，改变的行为是`compiletest`期望在测试中调用的 header 命令定义的失败代码，而不是默认值。

虽然具体到`failure-status`（因为每个头命令都有不同的实现来调用行为更改）也许看一个案例的行为改变实现是很有帮助的，仅作为一个例子。实施`failure-status`，`check_correct_failure_status()`找到的功能`TestCx`实施块，位于[SRC /工具/ compiletest / src 目录/ runtest.rs](https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/runtest.rs)，被修改如下：

```diff
@@ -295,11 +295,14 @@ impl<'test> TestCx<'test> {
     }

     fn check_correct_failure_status(&self, proc_res: &ProcRes) {
-        // The value the rust runtime returns on failure
-        const RUST_ERR: i32 = 101;
-        if proc_res.status.code() != Some(RUST_ERR) {
+        let expected_status = Some(self.props.failure_status);
+        let received_status = proc_res.status.code();
+
+        if expected_status != received_status {
             self.fatal_proc_rec(
-                &format!("failure produced the wrong error: {}", proc_res.status),
+                &format!("Error: expected failure status ({:?}) but received status {:?}.",
+                         expected_status,
+                         received_status),
                 proc_res,
             );
         }
@@ -320,7 +323,6 @@ impl<'test> TestCx<'test> {
         );

         let proc_res = self.exec_compiled_test();
-
         if !proc_res.status.success() {
             self.fatal_proc_rec("test run failed!", &proc_res);
         }
@@ -499,7 +501,6 @@ impl<'test> TestCx<'test> {
                 expected,
                 actual
             );
-            panic!();
         }
     }
```

注意使用`self.props.failure_status`访问 header 命令属性。在未指定 failure status header 命令的测试中，`self.props.failure_status`在撰写本文时，将评估为默认值 101。但是对于指定例如标题命令的测试，`// failure-status: 1`，`self.props.failure_status`将评估为 1，as`parse_failure_status()`会超越的`TestProps`默认值，特别针对该测试。

[test]: https://github.com/rust-lang/rust/tree/master/src/test
[header]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/header.rs
[common]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/common.rs
