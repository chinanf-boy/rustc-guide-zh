## 调试LLVM

> 注意：如果您正在寻找有关代码生成的信息，请参阅[本章][codegen]代替。

[codegen]: ../codegen.md

本节介绍如何在代码生成中调试编译器错误（例如，编译器生成某段代码或在LLVM中崩溃的原因）。LLVM本身就是一个大项目，可能需要有自己的调试文档（不是我能找到的）。但是这里有一些在rustc环境中很重要的提示：

作为一般规则，编译器通过分析代码生成大量信息。因此，有用的第一步通常是找到一个最小的例子。一种方法是

1.  创建一个可以重现问题的新箱子（例如，添加任何箱子作为依赖项，并从那里使用它）

2.  通过删除外部依赖关系来最小化包;也就是说，移动与新箱子相关的所有东西

3.  通过缩短代码来进一步减少问题（有一些工具可以帮助解决这个问题`creduce`）

官方编译器（包括nightlies）已禁用LLVM断言，这意味着LLVM断言失败可能会显示为编译器崩溃（不是ICE而是“真正的”崩溃）和其他类型的奇怪行为。如果您遇到这些问题，最好尝试使用启用了LLVM断言的编译器 - 每晚“alt”或您通过设置自己构建的编译器`[llvm] assertions=true`在你的config.toml中 - 看看是否有任何事情发生。

rustc构建过程将LLVM工具构建到其中`./build/<host-triple>/llvm/bin`。它们可以直接调用。

默认的rustc编译管道具有多个codegen单元，这些单元很难手动复制，并且意味着并行多次调用LLVM。如果你可以逃脱它（即如果它不会使你的bug消失），传递`-C codegen-units=1`生锈将使调试更容易。

要生成LLVM IR，需要通过`--emit=llvm-ir`旗。如果您是通过货物建造，请使用`RUSTFLAGS`环境变量（例如`RUSTFLAGS='--emit=llvm-ir'`）。这会导致rustc将LLVM IR吐出到目标目录中。

`cargo llvm-ir [options] path`为特定功能吐出LLVM IR`path`。（`cargo install cargo-asm`安装`cargo asm`和`cargo
llvm-ir`）`--build-type=debug`为调试生成发出代码。还有其他有用的选项。此外，llvm-ir中的调试信息会使输出混乱很多：`RUSTFLAGS="-C debuginfo=0"`真的很有用。

`RUSTFLAGS="-C save-temps"`编译期间在不同阶段输出llvm比特码（与ir不同），这有时很有用。只需将比特码文件转换为`.ll`文件使用`llvm-dis`它应该在rustc的目标本地编译中。

如果要使用优化管道，可以使用`opt`工具从`./build/<host-triple>/llvm/bin/`用Rustc发射的LLVM红外。请注意，根据是否`-O`是启用的，即使没有llvm的优化，因此如果要使用ir rustc发射器，您应该：

```bash
$ rustc +local my-file.rs --emit=llvm-ir -O -C no-prepopulate-passes \
    -C codegen-units=1
$ OPT=./build/$TRIPLE/llvm/bin/opt
$ $OPT -S -O2 < my-file.ll > my
```

如果只想在llvm管道期间获取llvm ir，例如查看哪个ir导致优化时间断言失败，或者查看llvm何时执行特定优化，则可以传递rustc标志。`-C
llvm-args=-print-after-all`，并可能添加`-C
llvm-args='-filter-print-funcs=EXACT_FUNCTION_NAME`（例如）`-C
llvm-args='-filter-print-funcs=_ZN11collections3str21_$LT$impl$u20$str$GT$\
7replace17hbe10ea2e7c809b0bE'`）

这会产生大量输出到标准错误中，所以您需要将其传输到某个文件中。另外，如果您两者都不使用`-filter-print-funcs`也不`-C
codegen-units=1`然后，由于多个codegen单元并行运行，打印输出将混合在一起，您将无法读取任何内容。

如果您只需要特定函数的IR（例如，您想知道它为什么会导致断言或不能正确优化），可以使用`llvm-extract`，例如

```bash
$ ./build/$TRIPLE/llvm/bin/llvm-extract \
    -func='_ZN11collections3str21_$LT$impl$u20$str$GT$7replace17hbe10ea2e7c809b0bE' \
    -S \
    < unextracted.ll \
    > extracted.ll
```

### 归档LLVM错误报告

在提交LLVM bug报告时，您可能需要某种演示问题的最小工作示例。Godbolt编译器资源管理器对此非常有用。

1.  一旦您对有问题的代码有了一些llvm ir（见上文），就可以用godbolt创建一个最小的工作示例。去[GCK.GordBo.Org](https://gcc.godbolt.org).

2.  选择`LLVM-IR`作为编程语言。

3.  使用`llc`将红外线编译为特定目标，如下所示：

    -   有一些有用的标志：`-mattr`启用目标功能，`-march=`选择目标，`-mcpu=`选择CPU等。
    -   类似命令`llc -march=help`输出所有可用的架构，这很有用，因为有时rust arch名称和llvm名称不匹配。
    -   如果您自己在某个地方编译了rustc，那么在目标目录中有用于`llc`，`opt`等。

4.  如果要优化llvm-ir，可以使用`opt`了解LLVM优化如何转换它。

5.  一旦你有了一个godbolt链接来演示这个问题，就很容易填写一个llvm错误。
