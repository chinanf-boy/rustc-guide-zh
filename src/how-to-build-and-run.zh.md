# 如何构建编译器并运行所构建的内容

编译器是使用一个名为`x.py`构建的。您需要安装 python 才能运行它。但在我们开始之前，如果你要 hacking 到`rustc`，您需要调整编译器的配置。默认配置是面向用户，而不是开发人员，去运行编译器。

### 创建 config.toml

开始，复制[`config.toml.example`]到`config.toml`：

[`config.toml.example`]: https://github.com/rust-lang/rust/blob/master/config.toml.example

```bash
> cd $RUST_CHECKOUT
> cp config.toml.example config.toml
```

然后您将希望打开该文件，并更改以下设置（可能还有其他设置，例如`llvm.ccache`）：

```toml
[llvm]
# 启用llvm断言，该断言将检查，由编译器生成的llvm 比特码
# 内部保持一致。这些对
# 编辑“codegen”特别有用。
assertions = true

[rust]
# 这会启用一些断言，但更重要的是，它会启用“调试”！`
# 调试rustc期间，所必需的记录宏会启用。
debug-assertions = true

# 这将使构建更为并行；它需要一些运行时
# 性能，(可能是减少内嵌代码），但它是值得的。
codegen-units = 0

# 我总是启用完整的debuginfo，尽管 debuginfo-lines 更重要。
debuginfo = true

# 为回溯提供行号。
debuginfo-lines = true
```

### 什么是 x.py？

x.py 是一个脚本，用于协调 rustc 存储库中的工具。它是可以构建文档、运行测试和编译 Rustc 的脚本。它现在是构建 rustc 的首选方法，它替换了以前的 makefile。以下是利用 x.py 有效处理的不同方法，解决存储库的各种常见任务。

### 运行 x.py 并构建 Stage1 编译器

要记住的一件事是`rustc`是一个*自举(bootstrapping)*编译器。也就是说，因为`rustc`是用 rust 编写的，我们需要使用旧版本的编译器来编译新版本。尤其是编译器的新版本，`libstd`和其他工具可能在内部使用一些不稳定的特性。导致的结果就是，需要分阶段完成`rustc`的编译：

- **第 0 阶段(Stage0)：**Stage0 编译器通常是当前的*beta*编译程序（`x.py`将为您下载）；虽然，您可以配置`x.py`使用其他东西。
- **第一阶段(Stage1)：**然后用 Stage0 编译器与代码克隆（新版本）编译，以生成 Stage1 编译器。但是，它是用一个旧的编译器（Stage0）构建的，所以为了优化 Stage1 编译器，我们将进入下一个阶段。
  - （理论上，阶段 1 编译器在功能上与阶段 2 编译器相同，但实际上存在细微的差异。尤其是，Stage1 编译器本身是由 Stage0 构建的，因此不是由工作目录中的源代码构建的：这意味着编译器源代码中使用的符号名可能与 Stage1 编译器所生成的符号名不匹配。这在使用动态链接（例如，with-derives）时很重要。这意味着有时，当使用阶段 1 运行时，某些测试不工作。）
- **第二阶段(Stage2)：**我们用 Stage1 编译器重新构建自身，以生成 Stage2 编译器（没错，自我构建），以拥有*最新优化*。（默认情况下，我们复制 Stage1 库，供 Stage2 编译器使用，因为它们应该是相同的。）
- _（可选）_ **第 3 阶段(Stage3)**：为了检查新编译器的健全性，我们可以使用 Stage2 编译器构建库。结果应该和以前一样，除非有什么东西坏了。

#### 构建标志

还有其他一些可以传递给 x.py 构建部分的标志，这些标志有助于缩短编译时间或适应可能需要更改的其他内容。他们是：

```bash
Options:
    -v, --verbose       use verbose output (-vv for very verbose)
    -i, --incremental   使用 增量 编译
        --config FILE   构建过程的 TOML格式 配置文件
        --build BUILD   stage0编译器的构建目标
        --host HOST     构建的主机目标
        --target TARGET 构建的 target 目标
        --on-fail CMD   失败时运行的命令
        --stage N       构建 阶段
        --keep-stage N  在不重新编译的情况下保持的阶段
        --src DIR       Rust checkout 之根路径
    -j, --jobs JOBS     并行运行的作业数
    -h, --help          print this help message
```

对要 hacking 的童鞋，通常构建阶段 1 编译器就足够了，但是对于最终的测试和发布，会使用阶段 2 编译器。

`./x.py check`构建 Rust 编译器的速度非常快。特别是，当您执行某种“基于类型的重构”（例如重命名方法或更改某个函数的签名）时，它非常有用。

<a name=command></a>

一旦创建了 config.toml，现在就可以运行了`x.py`。这里有很多选择，但是让我们从建立本地 Rust ，最好的“起步”命令开始：

```bash
> ./x.py build -i --stage 1 src/libstd
```

这*看*起来就像，它只构建 libstd 一样，但事实并非如此。此命令的作用如下：

- 使用 Stage0 编译器构建 libstd（使用增量）
- 使用 Stage0 编译器构建 librustc（使用增量）
  - 这将生成 Stage1 编译器
- 使用 Stage1 编译器生成 libstd（不能使用增量）

这最终产品（Stage1 编译器 + 使用该编译器构建的 libs）是构建其他 rust 程序所需要的。

注意，该命令包括`-i`开关。这将启用增量编译。这将用于加快进程的前两个步骤：特别是，如果您只是做了一个小的更改，我们应该能够使用您的旧结果，来让阶段 1 **编译器**的生成速度更快。

不幸的是，增量不能用来加速生成 Stage1 库。这是因为只有在运行*同一编译器*连续两次，增量才有用。在这种情况下，我们每一次都构建一个*新的阶段 1 编译器*。因此，旧的增量结果可能不适用。**因此，您可能会发现构建 Stage1 的 libstd 是您的瓶颈。**--但不要害怕，这里有一个（黑客）解决方案。参见下面的[“推荐工作流”部分](#workflow)。

注意，整个命令只提供了完整的 rustc 构建的一个子集。这个**完整的**Rust 建筑（`./x.py build`所做的）还有，接下来的很多步骤：

- 使用 Stage1 编译器构建 librustc 和 rustc。
  - 这里的结果编译器称为“阶段 2”编译器。
- 使用 Stage2 编译器构建 libstd。
- 使用 Stage2 编译器构建 librustdoc 和其他一些东西。

<a name=toolchain></a>

### 构建特定的组件

仅生成 libcore 库

```bash
> ./x.py build src/libcore
```

仅生成 libcore 和 libproc_macro 库

```bash
> ./x.py build src/libcore src/libproc_macro
```

仅构建第一阶段之前 libcore

```bash
> ./x.py build src/libcore --stage 1
```

有时，您可能只想测试正在处理的部分是否可以编译。使用这些命令，您可以在进行更大的构建之前测试它编译的代码，以确保它与编译器一起工作。如前所示，您还可以在末尾传递标志，如 `--stage`。

### 创建一个 rustup 工具链

一旦成功构建了 Rustc，您会`build`目录看到一堆文件。为了实际运行生成的 rustc，我们建议创建 rustup 工具链。第一个命令将运行 Stage1 编译器（我们在上面构建的）。第二个命令将执行 Stage2 编译器（我们没有构建它，但是您可能需要在某个时刻构建它；例如，如果您想运行整个测试套件）。

```bash
> rustup toolchain link stage1 build/<host-triple>/stage1
> rustup toolchain link stage2 build/<host-triple>/stage2
```

这个`<host-triple>`通常为以下之一：

- Linux：`x86_64-unknown-linux-gnu`
- Mac：`x86_64-apple-darwin`
- Windows：`x86_64-pc-windows-msvc`

现在你可以运行你建造的 rustc。如果你加上`-vV`，您应该看到一个版本号配以`-dev`，表明是从本地环境生成：

```bash
> rustc +stage1 -vV
rustc 1.25.0-dev
binary: rustc
commit-hash: unknown
commit-date: unknown
host: x86_64-unknown-linux-gnu
release: 1.25.0-dev
LLVM version: 4.0
```

<a name=workflow></a>

### 加快编译器构建的建议工作流

有两个工作流可用于更快地构建编译器。

**检查，检查，然后再次 check。**第一个工作流在执行简单重构时很有用，它是不断地运行`./x.py check`。在这里，您只是检查编译器是否可以**构建**，但通常这就是您所修改后（例如，重命名方法时），需要的全部。之后当您实际需要运行测试时，可以用`./x.py build`。

事实上，有时推迟测试是有用的，因为你不能百分之百地确定代码会工作。然后，您可以继续构建重构提交，并且只在稍后运行测试。然后你可以使用`git bisect`**精确地**追踪哪个提交导致了问题。这种风格的一个好处是在末尾，会留下一组相当仔细的提交，所有这些提交都是构建并通过测试的。这通常有助于审查。

**`--keep-stage`的增量构建。**有时仅仅检查编译器的是否构建，远远不够。一个常见的例子是，您需要添加`debug!`语句来检查某些状态的值，或更好地理解某问题。在这种情况下，您真的需要一个完整的构建。而这时，通过增量，您通常可以非常快地完成这些构建（例如，大约 30 秒）：唯一的问题是，这需要一点捏造，可能会构建不起作用的编译器（但很容易检测到并修复）。

您需要的命令序列如下：

- 初始版本：`./x.py build -i --stage 1 src/libstd`
  - 如[上述的文档](#command)所说，这将构建一个阶段 1 的功能性编译器
- 后续版本：`./x.py build -i --stage 1 src/libstd --keep-stage 1`
  - 注意，我们添加了`--keep-stage 1`标志

`--keep-stage 1`的影响，是*假定*旧的标准库可以重复使用。如果您正在编写编译器，这几乎总是正确的：毕竟您没有更改标准库。但有些情况会不对路：例如，如果您正在编辑编译器的“元数据(metadata)”部分，该部分控制编译器，如何将类型和其他状态编码到`rlib`文件，或者如果您正在编辑元数据中出现的内容（例如 MIR 的定义）。

**TL;DR 是在使用时，您可能会从`--keep-stage 1`编译中获得奇怪的行为。**--例如，奇怪[ICEs](appendix/glossary.zh.html)或者其他恐慌。在这种情况下，您只需从命令行中删除`--keep-stage 1`，重新构建。这应该能解决问题。

您也可以使用`--keep-stage 1`运行测试时。像这样：

- 初次测试运行：`./x.py test -i --stage 1 src/test/ui`
- 后续测试运行：`./x.py test -i --stage 1 src/test/ui --keep-stage 1`

### 其他 x.py 命令

下面是一些其他有用的 x.py 命令。我们将在其他部分详细介绍其中的一些内容：

- 建筑材料：
  - `./x.py clean`–清除生成目录（`rm -rf build`也可以，但是你必须重建 llvm）
  - `./x.py build --stage 1`–使用阶段 1 编译器构建所有内容，而不仅仅是 libstd
  - `./x.py build`–构建 Stage2 编译器
- 运行测试（参见[运行测试部分](./tests/running.zh.html)有关详细信息：
  - `./x.py test --stage 1 src/libstd`运行来自 libstd 的`#[test]`测试
  - `./x.py test --stage 1 src/test/run-pass`运行`run-pass`测试套件

### 插件

Rustc 的一个挑战是 RLS 不能处理它，因为它是一个引导编译器。这使得代码导航变得困难。一种解决方案是使用`ctags`. 可以使用以下脚本进行设置：[https://github.com/nikomatasakis/rust-etags][etags].

ctags 很容易集成到 emacs 和 vim 中。然后可以使用以下内容来构建和生成标记：

```console
$ rust-ctags src/lib* && ./x.py build <something>
```

这允许您使用上次构建时，所使用的任何函数执行“跳转到 def”，这非常有用。

[etags]: https://github.com/nikomatsakis/rust-etags

### 清除生成目录

有时你需要重新开始，但通常情况下不是这样。如果您需要运行这个程序，那么 Rust 的构建(rustbuild) 很可能不会正常运行，您应该提交一个 bug，说明出了什么问题。如果你确实需要清理所有的东西，那么你只需要运行一个命令！

```bash
> ./x.py clean
```

### 编译器文档

有关 rust 组件的文档，请参见[rustc doc].

[rustc doc]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/
