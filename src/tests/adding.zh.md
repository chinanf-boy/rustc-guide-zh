# 添加新测试

**一般来说，我们期望每一个修复 rustc 中 bug 的 pr 都伴随着某种回归测试。**这项测试应该在硕士学位考试中失败，但在公共关系考试之后通过。这些测试对于防止我们重复过去的错误是非常有用的。

要添加新的测试，您通常要做的第一件事是创建一个文件，通常是一个锈源文件。测试文件具有特定的结构：

- 他们应该吃点[解释测试内容的评论](#explanatory_comment)；
- 接下来，他们可以有一个或多个[头命令](#header_commands)，这是测试解释器知道如何解释的特殊注释。
- 最后，他们有生锈的来源。这可能有多种[错误注释](#error_annotations)指示预期的编译错误或警告。

根据测试套件的不同，可能还需要注意其他一些细节：

- 为了[这个`ui`测试套件](#ui)，需要生成引用输出文件。

## 我应该增加什么样的测试？

很难知道要使用哪种测试。以下是一些粗略的启发式方法：

- 有些测试有专门的需求：
  - 需要运行 gdb 或 lldb 吗？使用`debuginfo`测试套件
  - 需要检查 llvm-ir 或 mir-ir 吗？使用`codegen`或`mir-opt`测试套件
  - 需要运行 RustDoc 吗？喜欢一个`rustdoc`测试
  - 需要以某种方式检查生成的二进制文件吗？然后使用`run-make`
- 对于大多数其他事情，[一`ui`（或`ui-fulldeps`）测试](#ui)首选：
  - `ui`测试包括运行通过、编译失败和分析失败测试
  - 如果出现警告或错误，`ui`测试捕获完整的输出，这使得检查更容易，但也有助于防止输出中的“隐藏”回归。

## 命名测试

传统上，我们在测试的名称中没有太多的结构。而且，在很长一段时间内，rustc 测试运行程序不支持子目录（它现在支持），因此测试套件[`src/test/run-pass`]里面有一大堆文件。这不是理想的设置。

[`src/test/run-pass`]: https://github.com/rust-lang/rust/tree/master/src/test/run-pass/

对于回归测试——基本上是一些来自互联网的随机代码片段——我们通常只是在问题之后命名测试。例如，`src/test/run-pass/issue-12345.rs`.但是，如果可能的话，最好将测试放在一个目录中，该目录有助于确定在这里测试的是哪段代码（例如，`borrowck/issue-12345.rs`或者给它取一个更有意义的名字。不过，**一定要在某个地方包括问题编号**.

编写新功能时，**创建一个子目录来存储测试**. 例如，如果您正在实现 RFC1234（“小部件”），那么将测试放在如下目录中可能是有意义的：

- `src/test/ui/rfc1234-widgets/`
- `src/test/run-pass/rfc1234-widgets/`
- 等

在其他情况下，可能已经有了一个合适的目录。（要使用的正确目录结构实际上是一个活跃争论的领域。）

<a name="explanatory_comment"></a>

## 解释测试内容的评论

创建测试文件时，**在文件开头包含总结测试点的注释**. 这应该突出显示测试的哪些部分更重要，测试修复的 bug 是什么。引用问题编号通常非常有用。

这个评论不一定要非常广泛。就像“18060 年的回归测试：匹配臂的匹配顺序是错误的。”可能已经足够了。

这些注释对于以后的测试中断时的其他人非常有用，因为它们通常可以突出显示问题所在。如果出于某种原因需要重构测试，它们也很有用，因为它们让其他人知道测试的哪些部分很重要（通常必须重写测试，因为它不再测试要测试的内容，然后知道要测试的内容很有用*是*旨在精确测试）。

<a name="header_commands"></a>

## 头命令：配置 rustc

头命令是测试运行程序知道如何解释的特殊注释。它们必须出现在试验中的锈源之前。它们通常放在解释测试要点的简短注释之后。例如，此测试使用`// compile-flags`用于指定在编译测试时要提供给 rustc 的自定义标志的命令：

```rust,ignore
// Test the behavior of `0 - 1` when overflow checks are disabled.

// compile-flags: -Coverflow-checks=off

fn main() {
    let x = 0 - 1;
    ...
}
```

### 忽略的测试

这些用于在某些情况下忽略测试，这意味着不会编译或运行测试。

- `ignore-X`在哪里？`X`是目标细节还是阶段将忽略相应的测试（见下文）
- `only-X`就像`ignore-X`，但会*只有*在该目标或阶段上运行测试
- `ignore-pretty`不会编译漂亮的打印测试（这样做是为了测试漂亮的打印机，但可能并不总是有效）
- `ignore-test`总是忽略测试
- `ignore-lldb`和`ignore-gdb`将跳过该调试器上的 debuginfo 测试。
- `ignore-gdb-version`当使用某些 gdb 版本时，可用于忽略测试

一些例子`X`在里面`ignore-X`以下内容：

- 架构：`aarch64`，请`arm`，请`asmjs`，请`mips`，请`wasm32`，请`x86_64`，请`x86`，…
- 操作系统：`android`，请`emscripten`，请`freebsd`，请`ios`，请`linux`，`macos`，`windows`，…
- 环境（目标三的第四个字）：`gnu`，`msvc`，`musl`.
- 指针宽度：`32bit`，`64bit`.
- 阶段：`stage0`，`stage1`，`stage2`.

### 其他头命令

这是其他头命令的列表。这份清单并不详尽。头命令通常可以通过浏览`TestProps`结构发现于[`header.rs`]来自 compiletest 源。

- `run-rustfix`对于 UI 测试，指示测试生成结构化建议。测试编写器应创建一个`.fixed`文件，其中包含应用建议的源。运行测试时，compiletest 首先检查是否生成正确的 lint/warning。然后，它应用建议并与`.fixed`（必须匹配）。最后，编译固定源代码，并要求编译成功。这个`.fixed`文件也可以自动生成`--bless`选项，已讨论[在下面](#bless).
- `min-gdb-version`指定此测试所需的最低 gdb 版本；另请参见`ignore-gdb-version`
- `min-lldb-version`指定此测试所需的最低 LLDB 版本
- `rust-lldb`使测试的 lldb 部分仅在使用的 lldb 包含 rust 插件时运行
- `no-system-llvm`如果使用系统 llvm，则会导致忽略测试
- `min-llvm-version`指定此测试所需的最低 llvm 版本
- `min-system-llvm-version`指定此测试所需的最低系统 llvm 版本；如果系统 llvm 正在使用且不满足最低版本，则忽略此测试。当 LLVM 功能被反向移植以信任 LLVM 时，这很有用。
- `ignore-llvm-version`可用于在使用某些 LLVM 版本时跳过测试。这需要一个或两个参数；第一个参数是要忽略的第一个版本。如果没有给出第二个参数，则忽略所有后续版本；否则，第二个参数是要忽略的最后一个版本。
- `compile-pass`对于 UI 测试，指示测试应该编译，而不是默认测试应该出错。
- `compile-flags`将额外的命令行参数传递给编译器，例如`compile-flags -g`强制启用 debuginfo。
- `should-fail`指示测试应该失败；用于“元测试”，在该测试中，我们测试编译器程序本身，以检查它是否会在适当的场景中生成错误。对于漂亮的打印机测试，此标题将被忽略。
- `gate-test-X`在哪里？`X`功能是否将测试标记为功能 X 的“gate test”。此类测试应确保在没有正确的`#![feature(X)]`标签。每个不稳定的 lang 特性都需要进行一个 gate 测试。

[`header.rs`]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/header.rs

<a name="error_annotations"></a>

## 错误注释

错误注释指定编译器应发出的错误。它们被“附加”到错误所在的源代码行。

- `~`：将以下错误级别和消息与当前行关联
- `~|`：将以下错误级别和消息与上一条注释的行相关联
- `~^`：将以下错误级别和消息与前一行关联。每种插入符号`^`）你添加了一行，所以`~^^^^^^^`是七排吗？

您可以拥有的错误级别是：

1.  `ERROR`
2.  `WARNING`
3.  `NOTE`
4.  `HELP`和`SUGGESTION`\*

\* **注释**：`SUGGESTION`必须紧跟其后`HELP`.

## 修订

某些测试类支持“修订”（截至本文撰写之时，这包括运行通过、编译失败、运行失败和增量测试，尽管增量测试有些不同）。修订允许将单个测试文件用于多个测试。这是通过在文件顶部添加一个特殊的头来完成的：

```rust
// revisions: foo bar baz
```

这将导致测试被编译（和测试）三次，一次`--cfg foo`一次`--cfg bar`一次`--cfg baz`. 因此，您可以使用`#[cfg(foo)]`在测试中调整每个结果。

您还可以自定义标题和特定修订版的预期错误消息。为此，添加`[foo]`（或）`bar`，`baz`等）之后`//`评论，就像这样：

```rust
// A flag to pass in only for cfg `foo`:
//[foo]compile-flags: -Z verbose

#[cfg(foo)]
fn test_foo() {
    let x: usize = 32_u32; //[foo]~ ERROR mismatched types
}
```

请注意，并非所有标题在自定义为修订时都有意义。例如，`ignore-test`标题（以及所有“忽略”标题）当前仅适用于整个测试，而不适用于特定的修订。当定制为修订版时，唯一要真正工作的报头是错误模式和编译器标志。

<a name="ui"></a>

## UI 测试指南

UI 测试旨在捕获编译器的完整输出，以便我们可以测试表示的所有方面。它们通过编译文件（例如，[`ui/hello_world/main.rs`][hw-main]，捕获输出，然后应用一些规范化（见下文）。然后将此标准化结果与名为`ui/hello_world/main.stderr`和`ui/hello_world/main.stdout`. 如果这些文件中的任何一个不存在，则输出必须为空（这实际上是[这个特殊的测试][hw]）如果测试运行失败，我们将打印出当前输出，但它也保存在`build/<target-triple>/test/ui/hello_world/main.stdout`（此路径作为测试失败消息的一部分打印），因此您可以运行`diff`诸如此类。

[hw-main]: https://github.com/rust-lang/rust/blob/master/src/test/ui/hello_world/main.rs
[hw]: https://github.com/rust-lang/rust/blob/master/src/test/ui/hello_world/

### 不会导致编译错误的测试

默认情况下，需要 UI 测试**不编译**（在这种情况下，它应该至少包含一个`//~ ERROR`注释）。但是，您也可以在预期编译成功的地方进行 UI 测试，甚至可以运行生成的程序。只需添加以下内容之一[头命令](#header_commands)：

- `// compile-pass`–编译应成功，但不运行生成的二进制文件
- `// run-pass`–编译应该成功，我们应该运行生成的二进制文件

<a name="bless"></a>

### 编辑和更新参考文件

如果您有意更改编译器的输出，或者正在进行新的测试，则可以通过`--bless`测试子命令。例如，如果在`src/test/ui`失败了，你可以跑

```text
./x.py test --stage 1 src/test/ui --bless
```

自动调整`.stderr`，`.stdout`或`.fixed`所有测试的文件。当然，您也可以使用`--test-args your_test_name`标记，就像运行测试时一样。

### 归一化

应用的规范化旨在消除平台之间的输出差异，主要是关于文件名：

- 测试目录替换为`$DIR`
- 所有反斜杠（`\`）转换为正斜杠（`/`（对于 Windows）
- 所有 CR LF 换行都转换为 LF

有时，这些内置的规范化还不够。在这种情况下，您可以使用 header 命令提供自定义规范化规则，例如

```rust
// normalize-stdout-test: "foo" -> "bar"
// normalize-stderr-32bit: "fn\(\) \(32 bits\)" -> "fn\(\) \($$PTR bits\)"
// normalize-stderr-64bit: "fn\(\) \(64 bits\)" -> "fn\(\) \($$PTR bits\)"
```

这告诉测试，在 32 位平台上，每当编译器写`fn() (32 bits)`对于 stderr，应该将其规范化为读取`fn() ($PTR bits)`相反。类似于 64 位。替换由 regex 使用由提供的默认 regex 风格执行`regex`机箱。

相应的参考文件将使用规范化的输出来测试 32 位和 64 位平台：

```text
...
   |
   = note: source type: fn() ($PTR bits)
   = note: target type: u16 (16 bits)
...
```

请看[`ui/transmute/main.rs`][mrs]和[`main.stderr`][]具体用法示例。

[mrs]: https://github.com/rust-lang/rust/blob/master/src/test/ui/transmute/main.rs
[`main.stderr`]: https://github.com/rust-lang/rust/blob/master/src/test/ui/transmute/main.stderr

此外`normalize-stderr-32bit`和`-64bit`可以使用任何目标信息或支持的阶段`ignore-X`这里也是（例如`normalize-stderr-windows`或者简单地`normalize-stderr-test`无条件替换）。
