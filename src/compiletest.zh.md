# `compiletest`

## 介绍

`compiletest`是 Rust 测试套件的主要测试工具。它允许测试作者组织管理大量的测试（Rust 编译器有数千个），高效的测试执行（支持并行），并允许测试作者配置个人和测试组的行为和预期结果。

`compiletest`测试可以检查测试代码是否成功，失败或在某些情况下甚至是编译失败。测试通常被写成 Rust 源文件，其中在测试代码之前，或许会有注释命令，用来指导`compiletest`是否或如何运行测试，要期望什么行为，等等。如果您不熟悉编译器测试框架，请参阅[本章](./tests/intro.zh.html)了解更多背景资料

测试本身通常（但不总是）组织成“套件” - 例如，`run-pass`，一个表示测试应该成功的文件夹，`run-fail`，一个表示测试应该成功编译，但是返回一个失败（非零状态）的文件夹，`compile-fail`，一个表示测试应该无法编译的文件夹，还有更多。各种套件的定义都在[src/tools/compiletest/src/common.rs][common]的`pub struct Config`声明里面。有关编译器测试的不同套件的详细介绍，请参阅[添加新测试](./tests/adding.zh.html)。

## 添加一个新的测试文件

简而言之，只需在[src/test][test]下，适当的位置，简单创建新测试。不需要注册测试文件，`compiletest`将递归扫描[src/test][test]的子文件夹，并执行找到的任何 Rust 测试文件。看到[`Adding new tests`](./tests/adding.zh.html)有关如何添加新测试的完整指南。

## 头命令

源文件注释出现在源文件顶部附近的注释中，在任何测试的代码*之前*都称为头(Header)命令。这些命令的指示有很多，不如让`compiletest`忽略这个测试，设定对预期是否会成功编译的要求，或预期测试的返回代码是什么。头命令（及其对应的内联项，错误信息命令）在[这里](./tests/adding.zh.html#header-commands-configuring-rustc)有更全面的描述。

### 添加新的头命令

头命令在[src/tools/compiletest/src/header.rs][header]的`TestProps`结构中定义。对于较高的层次，这里定义了许多测试属性，所有测试属性都在`TestProp`结构的`impl`块中，设置为默认值。任何测试都可以，将有问题的属性用头命令注释（`//`）来覆盖此默认值，有效位置在测试源文件中和在任何源代码之前。

#### 使用 头命令

这是一个例子，指定`must-compile-successfully`头命令，不带参数，后跟`failure-status`头命令，它接受一个参数（在这种情况下，值为 1）。`failure-status`能让`compiletest`期望失败状态码为 1（而不是当前的 Rust 默认值 101，本文撰写时，）头命令和参数列表（如果有的话）通常用冒号分隔：

```rust,ignore
// must-compile-successfully
// failure-status: 1

#![feature(termination_trait)]

use std::io::{Error, ErrorKind};

fn main() -> Result<(), Box<Error>> {
    Err(Box::new(Error::new(ErrorKind::Other, "returned Box<Error> from main()")))
}
```

#### 添加新的头命令属性

如果需要在逐个测试的基础上，定义一些测试属性或行为，则可以添加新的头命令。头命令属性在运行时，作为头命令的后备（保存命令的当前值）。

要添加新的头命令属性：

- 1。查找在[src/tools/compiletest/src/header.rs][header]文件中的`pub struct TestProps`声明，并将新的公共属性添加到声明的末尾。
- 2.寻找结构声明后，紧跟着的`impl TestProps`实现块，并为新属性初始化默认值。

#### 添加新的头命令解析器

当`compiletest`遇到一个测试文件，它通过调用在`Config`struct 的实现块中，定义的每个解析器，一次解析一行文件。[src/tools/compiletest/src/header.rs][header]中也有定义。（注意`Config`结构的声明块可以在[src/tools/compiletest/src/common.rs][common]。`TestProps`的`load_from()`方法将尝试将当前文本行传递给每个解析器，然后，常会检查该行是否以特定注释（`//`）开头，如头命令`// must-compile-successfully`或者是`// failure-status`。注释标记后面的空格是可选的，为了美观。

解析器能覆盖给出头命令属性的默认值，一种做法是，通过在测试文件中指定头命令；另一种是，通过在测试文件中指定参数值，具体取决于头命令。

解析器定义于`impl Config`，典型的命名方式`parse_<header_command>`（注意烤肉串形式{kebab-case}`<header-command>`转变为蛇形式{snake-case}`<header_command>`）。`impl Config`还定义了几个“低级”解析器，这让解析常见模式变得简单，像：存在与否的（`parse_name_directive()`）；header-command:parameter(s)的
(`parse_name_value_directive()`)；或是特定的`cfg`属性已定义，可选性解析的（`has_cfg_prefix()`）；还有很多。低级解析器位于`impl Config`块末尾附近; 请务必仔细查看它们及其上面的相关解析器，看看它们是如何被用来避免，编写额外非必要的解析代码的。

一个具体的例子，这里是`parse_failure_status()`解析器实现，在[src/tools/compiletest/src/header.rs][header]：

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

## 实现行为的改变

当一个测试调用特定的头命令时，预计会有些行为将因此而改变。显然，行为的变化取决于头命令的目标。如果是`failure-status`，改变的行为是，`compiletest`预期的失败代码，由测试所调用的头命令定义，而不是默认值决定。

虽然仅具体到`failure-status`（因为每个头命令都有不同的实现，以此导致行为的改变），但也许看看一个案例的行为改变实现过程，对你有所帮助，这里仅作为一个例子给你参考：要实现`failure-status`，先在`TestCx`实现块中找到`check_correct_failure_status()`函数，它位于[src/tools/compiletest/src/runtest.rs](https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/runtest.rs)，被修改如下：

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

注意使用`self.props.failure_status`访问头命令属性。在未指定 failure status 头命令的测试中，`self.props.failure_status`(撰写本文时)将定为默认值 101。但是对，有指定头命令的测试，例如`// failure-status: 1`，如此一来，`self.props.failure_status`变为 1，这样，`parse_failure_status()`就会覆盖`TestProps`默认值。结论仅针对该测试。

[test]: https://github.com/rust-lang/rust/tree/master/src/test
[header]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/header.rs
[common]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/common.rs
