# 如何构建编译器并运行所构建的内容

编译器是使用一个名为`x.py`. 您需要安装 python 才能运行它。但在我们开始之前，如果你要入侵`rustc`，您需要调整编译器的配置。默认配置面向作为用户而不是开发人员运行编译器。

### 创建 config.toml

开始，复制[`config.toml.example`]到`config.toml`：

[`config.toml.example`]: https://github.com/rust-lang/rust/blob/master/config.toml.example

```bash
> cd $RUST_CHECKOUT
> cp config.toml.example config.toml
```

然后您将希望打开该文件并更改以下设置（可能还有其他设置，例如`llvm.ccache`）：

```toml
[llvm]
# Enables LLVM assertions, which will check that the LLVM bitcode generated
# by the compiler is internally consistent. These are particularly helpful
# if you edit `codegen`.
assertions = true

[rust]
# This enables some assertions, but more importantly it enables the `debug!`
# logging macros that are essential for debugging rustc.
debug-assertions = true

# This will make your build more parallel; it costs a bit of runtime
# performance perhaps (less inlining) but it's worth it.
codegen-units = 0

# I always enable full debuginfo, though debuginfo-lines is more important.
debuginfo = true

# Gives you line numbers for backtraces.
debuginfo-lines = true
```

### 什么是 X.Py？

x.py 是用于协调 rustc 存储库中的工具的脚本。它是可以构建文档、运行测试和编译 RustC 的脚本。它现在是构建 rustc 的首选方法，它替换了以前的 makefile。以下是利用 x.py 有效处理各种常见任务的回购的不同方法。

### 运行 x.py 并构建 Stage1 编译器

要记住的一件事是`rustc`是一个*自举*编译程序。也就是说，因为`rustc`是用 rust 编写的，我们需要使用旧版本的编译器来编译新版本。尤其是编译器的新版本，`libstd`和其他工具可能在内部使用一些不稳定的特性。结果是编译`rustc`分阶段完成：

- **第 0 阶段：**Stage0 编译器通常是当前的*贝塔*编译程序（编译器）`x.py`将为您下载）；您可以配置`x.py`但是，使用其他东西。
- **第一阶段：**然后用 Stage0 编译器编译克隆（新版本）中的代码以生成 Stage1 编译器。但是，它是用一个旧的编译器（Stage0）构建的，所以为了优化 Stage1 编译器，我们将进入下一个阶段。
  - （理论上，阶段 1 编译器在功能上与阶段 2 编译器相同，但实际上存在细微的差异。尤其是，Stage1 编译器本身是由 Stage0 构建的，因此不是由工作目录中的源代码构建的：这意味着编译器源代码中使用的符号名可能与 Stage1 编译器所生成的符号名不匹配。这在使用动态链接（例如，with-derives）时很重要。有时这意味着当使用阶段 1 运行时，某些测试不工作。）
- **第二阶段：**我们用自身重新构建 Stage1 编译器，以生成 Stage2 编译器（即，它自己构建），以拥有*最新优化*.（默认情况下，我们复制 Stage1 库供 Stage2 编译器使用，因为它们应该是相同的。）
- _（可选）_ **第 3 阶段**：为了检查新编译器的健全性，我们可以使用 Stage2 编译器构建库。结果应该和以前一样，除非有什么东西坏了。

#### 生成标志

还有其他一些标记可以传递给 x.py 的构建部分，这些标记有助于缩短编译时间或适应可能需要更改的其他内容。他们是：

```bash
Options:
    -v, --verbose       use verbose output (-vv for very verbose)
    -i, --incremental   use incremental compilation
        --config FILE   TOML configuration file for build
        --build BUILD   build target of the stage0 compiler
        --host HOST     host targets to build
        --target TARGET target targets to build
        --on-fail CMD   command to run on failure
        --stage N       stage to build
        --keep-stage N  stage to keep without recompiling
        --src DIR       path to the root of the rust checkout
    -j, --jobs JOBS     number of jobs to run in parallel
    -h, --help          print this help message
```

对于黑客攻击，通常构建阶段 1 编译器就足够了，但是对于最终的测试和发布，使用阶段 2 编译器。

`./x.py check`构建 Rust 编译器的速度非常快。特别是，当您执行某种“基于类型的重构”（例如重命名方法或更改某个函数的签名）时，它非常有用。

<a name=command></a>

一旦创建了 config.toml，现在就可以运行了`x.py`.这里有很多选择，但是让我们从建立本地锈病最好的“去”命令开始：

```bash
> ./x.py build -i --stage 1 src/libstd
```

今年五月*看*就像它只构建 libstd 一样，但事实并非如此。此命令的作用如下：

- 使用 Stage0 编译器构建 libstd（使用增量）
- 使用 Stage0 编译器构建 librustc（使用增量）
  - 这将生成 Stage1 编译器
- 使用 Stage1 编译器生成 libstd（不能使用增量）

这个最终产品（使用该编译器构建的 Stage1 编译器+libs）是构建其他 rust 程序所需要的。

注意，该命令包括`-i`开关。这将启用增量编译。这将用于加快进程的前两个步骤：特别是，如果您做了一个小的更改，我们应该能够使用您的旧结果来生成阶段 1**编译器**更快。

不幸的是，增量不能用来加速生成 Stage1 库。这是因为只有在运行*同一编译器*连续两次。在这种情况下，我们正在建立一个*新的阶段 1 编译器*每一次。因此，旧的增量结果可能不适用。**因此，您可能会发现构建 Stage1libstd 是您的瓶颈。**--但不要害怕，这里有一个（黑客）解决方案。参见[“推荐工作流”部分](#workflow)下面。

注意，整个命令只提供了完整的 rustc 构建的一个子集。这个**满的**生锈的建筑（如果你只是说`./x.py build`）还有很多步骤：

- 使用 Stage1 编译器构建 librustc 和 rustc。
  - 这里的结果编译器称为“阶段 2”编译器。
- 使用 Stage2 编译器构建 libstd。
- 使用 Stage2 编译器构建 librustDoc 和其他一些东西。

<a name=toolchain></a>

### 生成特定组件

仅生成 libcore 库

```bash
> ./x.py build src/libcore
```

仅生成 libcore 和 libproc\_宏库

```bash
> ./x.py build src/libcore src/libproc_macro
```

仅在第一阶段之前构建 libcore

```bash
> ./x.py build src/libcore --stage 1
```

有时，您可能只想测试正在处理的部分是否可以编译。使用这些命令，您可以在进行更大的构建之前测试它编译的代码，以确保它与编译器一起工作。如前所示，您还可以在末尾传递标志，如--stage。

### 创建一个生锈的工具链

一旦成功构建了 RustC，您将在`build`目录。为了实际运行生成的 rustc，我们建议创建 rustup 工具链。第一个将运行 Stage1 编译器（我们在上面构建）。第二个将执行 Stage2 编译器（我们没有构建它，但是您可能需要在某个时刻构建它；例如，如果您想运行整个测试套件）。

```bash
> rustup toolchain link stage1 build/<host-triple>/stage1
> rustup toolchain link stage2 build/<host-triple>/stage2
```

这个`<host-triple>`通常为以下之一：

- Linux：`x86_64-unknown-linux-gnu`
- 雨衣：`x86_64-apple-darwin`
- 窗户：`x86_64-pc-windows-msvc`

现在你可以运行你建造的铁锈。如果你跑步`-vV`，您应该看到一个版本号以`-dev`，指示从本地环境生成：

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

### 加快编译器生成的建议工作流

有两个工作流可用于更快地构建编译器。

**检查，检查，然后再次检查。**第一个工作流在执行简单重构时很有用，它是运行`./x.py check`连续不断地。在这里，您只是检查编译器是否可以**建造**，但通常这就是您所需要的全部（例如，重命名方法时）。你就可以跑了`./x.py build`当您实际需要运行测试时。

事实上，有时推迟测试是有用的，即使你不能百分之百地确定代码会工作。然后，您可以继续构建重构提交，并且只在稍后运行测试。然后你可以使用`git bisect`追踪**精确地**哪个提交导致了问题。这种风格的一个好的副作用是在末尾留下一组相当细粒度的提交，所有这些提交都是构建并通过测试的。这通常有助于审查。

**增量生成`--keep-stage`.**有时仅仅检查编译器的构建是否不够。一个常见的例子是，您需要添加`debug!`语句来检查某些状态的值或更好地理解问题。在这种情况下，您真的需要一个完整的构建。但是，通过利用增量，您通常可以非常快地完成这些构建（例如，大约 30 秒）：唯一的问题是，这需要一点推诿，可能会生成不起作用的编译器（但很容易检测到并修复）。

您需要的命令序列如下：

- 初始版本：`./x.py build -i --stage 1 src/libstd`
  - 作为[上述文件](#command)，这将构建一个功能阶段 1 编译器
- 后续版本：`./x.py build -i --stage 1 src/libstd --keep-stage 1`
  - 注意，我们添加了`--keep-stage 1`在此处标记

影响`--keep-stage 1`我们只是*假定*旧的标准库可以重复使用。如果您正在编辑编译器，这几乎总是正确的：毕竟您没有更改标准库。但有时情况并非如此：例如，如果您正在编辑编译器的“元数据”部分，该部分控制编译器如何将类型和其他状态编码到`rlib`文件，或者如果您正在编辑元数据中出现的内容（例如 mir 的定义）。

**tl；dr 是在使用时，您可能会从编译中获得奇怪的行为。`--keep-stage 1`**--例如，奇怪[冰](appendix/glossary.html)或者其他恐慌。在这种情况下，您只需删除`--keep-stage 1`从命令中重新生成。这应该能解决问题。

您也可以使用`--keep-stage 1`运行测试时。像这样：

- 初次试运行：`./x.py test -i --stage 1 src/test/ui`
- 后续试运行：`./x.py test -i --stage 1 src/test/ui --keep-stage 1`

### 其他 x.py 命令

下面是一些其他有用的 x.py 命令。我们将在其他部分详细介绍其中的一些内容：

- 建筑材料：
  - `./x.py clean`–清除生成目录（`rm -rf build`也可以，但是你必须重建 llvm）
  - `./x.py build --stage 1`–使用阶段 1 编译器构建所有内容，而不仅仅是 libstd
  - `./x.py build`–构建 Stage2 编译器
- 运行测试（参见[运行测试部分](./tests/running.html)有关详细信息：
  - `./x.py test --stage 1 src/libstd`运行`#[test]`来自 libstd 的测试
  - `./x.py test --stage 1 src/test/run-pass`运行`run-pass`测试套件

### 插件

RustC 的一个挑战是 RLS 不能处理它，因为它是一个引导编译器。这使得代码导航变得困难。一种解决方案是使用`ctags`. 可以使用以下脚本进行设置：[https://github.com/nikomatasakis/rust-etags][etags].

ctags 很容易集成到 emacs 和 vim 中。然后可以使用以下内容来构建和生成标记：

```console
$ rust-ctags src/lib* && ./x.py build <something>
```

这允许您使用上次构建时所使用的任何函数执行“跳转到 def”，这非常有用。

[etags]: https://github.com/nikomatsakis/rust-etags

### 清除生成目录

有时你需要重新开始，但通常情况下不是这样。如果您需要运行这个程序，那么 RustBuild 很可能不会正常运行，您应该提交一个 bug，说明出了什么问题。如果你确实需要清理所有的东西，那么你只需要运行一个命令！

```bash
> ./x.py clean
```

### 编译器文档

有关防锈部件的文档，请参见[rustc doc].

[rustc doc]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/
