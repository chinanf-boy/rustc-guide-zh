# 添加新测试

**一般来说，我们期望每一个修复 rustc bug 的 pr ，都伴随着某种性能退步测试。**这项测试应该在 master 分支中失败，但在 PR 之后成功。这些测试对于防止我们，重复过去的错误是非常有用的。

要添加新的测试，您通常要做的第一件事是创建一个文件，通常是一个 Rust 代码源文件。测试文件具有特定的流程结构：

- 他们应该有一些[解释测试内容的注释](#explanatory_comment)；
- 接下来，他们可以有一个或多个[头 命令](#header_commands)，这是给测试解释器所用的特殊注释。
- 最后，他们有 Rust 源。这可能有多种[错误注释](#error_annotations)指示预期的编译错误或警告。

根据测试套件的不同，可能还需要注意其他一些细节：

- 对[这个`ui`测试套件](#ui)来说，你需要生成引用的输出文件。

## 我应该增加什么样的测试？

很难知道要使用哪种测试。以下是一些粗略的启发：

- 有些测试有专门的需求：
  - 需要运行 gdb 或 lldb 吗？使用`debuginfo`测试套件
  - 需要检查 LLVM-IR 或 MIR-IR 吗？使用`codegen`或`mir-opt`测试套件
  - 需要运行 rustdoc 吗？倾向一个`rustdoc`测试
  - 需要以某种方式检查生成的二进制文件吗？然后使用`run-make`
- 对于大多数其他事情，[一个`ui`（或`ui-fulldeps`）测试](#ui)是首选：
  - `ui`测试包括 run-pass, compile-fail, 和 parse-fail 测试
  - 如果出现警告或错误，`ui`测试捕获完整的输出，这使得检查更容易，但也有助于防止输出中，“隐藏”的性能退步。

## 命名测试

传统上，我们在测试的名称上，没有太多要求。而且，在很长一段时间内，rustc 测试运行程序不支持子目录（它现在支持），因此像[`src/test/run-pass`]之类的测试套件，目录里面有一大堆文件。这不是理想的设置。

[`src/test/run-pass`]: https://github.com/rust-lang/rust/tree/master/src/test/run-pass/

对于性能退步测试 —— 基本上是一些来自互联网的随机代码片段 —— 我们通常只是在出现问题之后命名测试。例如，`src/test/run-pass/issue-12345.rs`，但是，如果可能的话，最好将测试放在一个目录中，这样有助于确定，要测试的是哪段代码（例如，`borrowck/issue-12345.rs`这样的，有个名称目录更好)或者给它取一个更有意义的名字。不过，**一定要在某个地方加上问题编号**.

编写新功能时，**创建一个子目录来存储测试**。 例如，如果您正在实现 RFC1234（“小部件”），那么将测试放在如下目录中，意思更明显：

- `src/test/ui/rfc1234-widgets/`
- `src/test/run-pass/rfc1234-widgets/`
- 等

在其他情况下，可能已经有了一个合适的目录。（正确目录结构的使用问题，实际上是一个兵家必争之地。）

<a name="explanatory_comment"></a>

## 解释测试内容的注释

创建测试文件时，**在文件开头包含，测试要点的总结注释**。 这应该突出显示测试的哪些部分更重要，本测试要修复的 bug 是什么。引用问题编号通常非常有用。

这个注释不一定要多详细。就像“#18060 的性能退步测试：match 语句的匹配顺序是错误的。”，这样可能已经足够了。

以后若你的测试坏了，这些注释对其他人会有用，因为它们通常可以突出问题所在。如果出于某种原因需要重构测试，它们也很有用，因为它们让其他人知道测试的哪些部分很重要（通常必须重写测试，因为它不再测试要测试的内容，而知道*原来*的测试内容，会对测试的精确性起到作用）。

<a name="header_commands"></a>

## 头命令：配置 rustc

头命令是给测试运行程序使用的特殊注释。它们必须出现在测试的 Rust 代码之前。它们通常放在解释测试要点的简短注释之后。例如，此测试使用了`// compile-flags`，表明在编译测试时，要提供给 rustc 的自定义标志的命令：

```rust,ignore
// 当溢出检查禁用，测试  `0 - 1` 的 行为，.

// compile-flags: -Coverflow-checks=off

fn main() {
    let x = 0 - 1;
    ...
}
```

### 忽略某测试

这些用于在一些情况下，忽略某测试，这意味着不会编译或运行某测试。

- `ignore-X`，这里的`X`，指的是，在目标细节还是哪阶段时，忽略相应的测试（见下文）
- `only-X`就像`ignore-X`，但*只会*在该目标或阶段上运行测试
- `ignore-pretty`不会编译 pretty-printed 测试（这样做是为了测试漂 pretty-printed，但可能并不总是有效）
- `ignore-test`总是忽略测试
- `ignore-lldb`和`ignore-gdb`将跳过该调试器上的 debuginfo 测试。
- `ignore-gdb-version`当某些 gdb 版本时，用来忽略测试

`ignore-X`的，一些`X`例子：

- 架构：`aarch64`，`arm`，`asmjs`，`mips`，`wasm32`，`x86_64`，`x86`，…
- 操作系统：`android`，`emscripten`，`freebsd`，`ios`，`linux`，`macos`，`windows`，…
- 环境（目标三的第四个字）：`gnu`，`msvc`，`musl`.
- 指针宽度：`32bit`，`64bit`.
- 阶段：`stage0`，`stage1`，`stage2`.

### 其他头命令

这是其他头命令的列表。这份清单并不详尽。详尽的头命令通常可以在 compiletest 源代码，[`header.rs`]文件的`TestProps`结构浏览。

- `run-rustfix`给 UI 测试，指示测试生成结构化建议。测试编写人应创建一个`.fixed`文件，其中包含应用的建议。运行测试时，compiletest 首先检查是否生成正确的 lint/warning。然后，它应用建议并与`.fixed`（必须匹配）。最后，编译 fixed 的源代码，并要求编译成功。这个`.fixed`文件也可以用`--bless`选项自动生成，[在下面](#bless)已有讨论。
- `min-gdb-version`指定此测试所需的最低 gdb 版本；另请参见`ignore-gdb-version`
- `min-lldb-version`指定此测试所需的最低 LLDB 版本
- `rust-lldb`限制测试的 lldb 部分，仅在使用的 lldb 包含 rust 插件时运行。
- `no-system-llvm`如果使用了系统 llvm，则会导致忽略测试
- `min-llvm-version`指定此测试所需的最低 llvm 版本
- `min-system-llvm-version`指定此测试所需的最低系统 llvm 版本；如果系统 llvm 正在使用且不满足最低版本，则忽略此测试。当 LLVM 功能被反向移植给 rust-LLVM 时，会很有用。
- `ignore-llvm-version`可用于，在使用某些 LLVM 版本时，跳过测试。这需要一个或两个参数；第一个参数是要忽略的第一个版本。如果没有给出第二个参数，则忽略所有后续版本；否则，第二个参数是要忽略的最后一个版本。
- `compile-pass`给 UI 测试，指明测试应该编译，而不是测试应该出错的默认值。
- `compile-flags`将额外的命令行参数传递给编译器，例如`compile-flags -g`能强制启用 debuginfo。
- `should-fail`指示测试应该失败；用于“元测试”，在该测试中，我们测试编译器程序本身，以检查它是否会在适当的场景中生成错误。对于 pretty-printer 测试，此标题将被忽略。
- `gate-test-X`，这里的`X`是一个给功能 X ，将测试标记为“gate test” 的。此类测试应确保，在没有正确的`#![feature(X)]`标签的情况下，尝试使用一个已 gate 的功能特性，编译器会给出错误。每个不稳定的 语言特性都需要进行一个 gate 测试。

[`header.rs`]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest/src/header.rs

<a name="error_annotations"></a>

## 错误注释

错误注释指定编译器应发出的错误。它们被“附加”到错误所在的源代码行。

- `~`：将以下错误级别和消息与当前行关联
- `~|`：将以下错误级别和消息与上一条注释的行相关联
- `~^`：将以下错误级别和消息与前一行关联。每插入（`^`）符号你添加了一行，所以`~^^^^^^^`是七行。

您可以拥有的错误级别是：

1.  `ERROR`
2.  `WARNING`
3.  `NOTE`
4.  `HELP`和`SUGGESTION`\*

\* **注释**：`SUGGESTION`必须紧跟其后`HELP`.

## 复习(revisions)

某些测试类支持“复习”（截至本文撰写之时，这包括 run-pass, compile-fail, run-fail, 和增量测试，尽管增量测试有些不同）。复习允许将单个测试文件用于多个测试。这是通过在文件顶部添加一个特殊的头来完成的：

```rust
// revisions: foo bar baz
```

这将导致测试被编译（和测试）三次，一次`--cfg foo`，一次`--cfg bar`，一次`--cfg baz`。因此，您可以使用`#[cfg(foo)]`在测试中调整每个结果。

您还可以自定义标题和，特定复习的预期错误消息。为此，在`//`注释之后添加`[foo]`（或`bar`，`baz`等），就像这样：

```rust
// A flag to pass in only for cfg `foo`:
//[foo]compile-flags: -Z verbose

#[cfg(foo)]
fn test_foo() {
    let x: usize = 32_u32; //[foo]~ ERROR mismatched types
}
```

请注意，并非所有标题在自定义为复习时都有意义。例如，当前`ignore-test`标题（以及所有“忽略”标题）仅适用于整个测试，而不适用于特定的复习。当定制一个复习时，唯一真正要工作的头是错误模式和编译器标志。

<a name="ui"></a>

## UI 测试指南

UI 测试旨在捕获编译器的完整输出，以便我们可以测试表达的所有方面。它工作方式是，通过编译文件（例如，[`ui/hello_world/main.rs`][hw-main]，捕获输出，然后应用一些规范化（见下文）。然后将此标准化结果与名为`ui/hello_world/main.stderr`和`ui/hello_world/main.stdout`的参考文件做比较。如果这些文件中的任何一个不存在，则输出必须为空（这实际上是[这个特殊的测试][hw]）如果测试运行失败，我们将打印出当前输出，但也将它保存在`build/<target-triple>/test/ui/hello_world/main.stdout`（此路径作为测试失败消息的一部分，打印出来），因此您可以运用`diff`诸如此类的工具。

[hw-main]: https://github.com/rust-lang/rust/blob/master/src/test/ui/hello_world/main.rs
[hw]: https://github.com/rust-lang/rust/blob/master/src/test/ui/hello_world/

### 不会导致编译错误的测试

默认情况下，一个 UI 测试需要**不能编译**（在这种情况下，它应该至少包含一个`//~ ERROR`注释）。但是，您也可以在预期编译成功的地方，进行 UI 测试，甚至可以运行生成的程序。只需添加以下[头命令](#header_commands)内容之一：

- `// compile-pass`–编译应成功，但不运行生成的二进制文件
- `// run-pass`–编译应该成功，我们应该运行生成的二进制文件

<a name="bless"></a>

### 编辑和更新参考文件

如果您有意更改编译器的输出，或者正在进行新的测试，则可以传递`--bless`选项给子测试命令。例如，如果在`src/test/ui`失败了，你可以运行

```text
./x.py test --stage 1 src/test/ui --bless
```

所有测试自动调整`.stderr`，`.stdout`或`.fixed`文件。当然，您也可以使用`--test-args your_test_name`标志，就像运行测试时一样。

### 规范化

应用的规范化，旨在消除平台之间的输出差异，主要是关于文件名：

- 测试目录替换为`$DIR`
- 所有反斜杠（`\`）转换为正斜杠（`/`（对 Windows）
- 所有 CR LF 换行，都转换为 LF

有时，这些内置的规范化还不够。在这种情况下，您可以使用 头命令提供自定义规范化规则，例如

```rust
// normalize-stdout-test: "foo" -> "bar"
// normalize-stderr-32bit: "fn\(\) \(32 bits\)" -> "fn\(\) \($$PTR bits\)"
// normalize-stderr-64bit: "fn\(\) \(64 bits\)" -> "fn\(\) \($$PTR bits\)"
```

这告诉测试，在 32 位平台上，每当编译器写`fn() (32 bits)`对于 stderr，应该将其规范化用`fn() ($PTR bits)`替换。类似于 64 位。替换由 正则式 执行，由`regex`箱提供的默认 regex 风格。

相应的参考文件，将使用规范化的输出来测试 32 位和 64 位平台：

```text
...
   |
   = note: source type: fn() ($PTR bits)
   = note: target type: u16 (16 bits)
...
```

请看[`ui/transmute/main.rs`][mrs]和[`main.stderr`][]，具体的用法示例。

[mrs]: https://github.com/rust-lang/rust/blob/master/src/test/ui/transmute/main.rs
[`main.stderr`]: https://github.com/rust-lang/rust/blob/master/src/test/ui/transmute/main.stderr

此外`normalize-stderr-32bit`和`-64bit`，可以使用任何目标的信息或这里，由`ignore-X`来支持阶段（例如`normalize-stderr-windows`或者简单地用`normalize-stderr-test`，进行无条件替换）。
