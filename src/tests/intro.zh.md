# 编译器测试框架

Rust项目运行各种不同的测试，由构建系统协调。（`x.py test`）测试编译器本身的主要测试工具是一个名为compiletest的工具（源代码位于[`src/tools/compiletest`]）本节简要概述了如何设置测试框架，然后详细介绍了[如何运行测试](./running.html#ui)以及[如何添加新测试](./adding.html).

[`src/tools/compiletest`]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest

## 编译器测试套件

编译器测试位于[`src/test`]目录。您将立即看到一系列子目录（例如`ui`，`run-make`等等）。每个目录都称为**测试套件**–它们包含一组以不同模式运行的测试。

[`src/test`]: https://github.com/rust-lang/rust/tree/master/src/test

以下是本文中测试套件的简要概述以及它们的含义。在某些情况下，测试套件链接到手册中提供更多详细信息的部分。

-   [`ui`](./adding.html#ui)–检查编译和/或运行测试的准确stdout/stderr的测试
-   `run-pass`–预期编译和执行成功的测试（无恐慌）
    -   `run-pass-valgrind`–应使用Valgrind进行的测试
-   `run-fail`–预期编译的测试，但在执行过程中会死机
-   `compile-fail`—预期编译失败的测试。
-   `parse-fail`–预期无法分析的测试
-   `pretty`–针对“漂亮的打印机”的测试，它从AST生成有效的锈代码。
-   `debuginfo`–在gdb或lldb中运行并查询调试信息的测试
-   `codegen`–测试编译并测试生成的LLVM代码，以确保我们想要的优化正在生效。
-   `mir-opt`–测试检查生成的mir的各个部分，以确保正确构建内容或执行预期的优化。
-   `incremental`–测试增量编译，检查在执行某些修改时，我们是否能够重用以前编译的结果。
-   `run-make`—基本上只执行`Makefile`灵活性的极限，但写起来很烦人。
-   `rustdoc`–对RustDoc进行测试，确保生成的文件包含预期的文档。
-   `*-fulldeps`–同上，但表明测试取决于除`libstd`（因此这些东西必须建造）

## 其他测试

Rust Build系统可处理各种其他事项的运行测试，包括：

-   **整洁的**–这是一个自定义工具，用于验证源代码样式和格式约定，例如拒绝长行。有更多的信息在[关于编码约定的章节](../conventions.html#formatting).

    例子：`./x.py test src/tools/tidy`

-   **单元测试**–铁锈标准库和许多铁锈包装包括典型的铁锈。`#[test]`单元测试。在引擎盖下，`x.py`将运行`cargo test`在每个包上运行所有测试。

    例子：`./x.py test src/libstd`

-   **文档测试**–Rust文档中嵌入的示例代码通过以下方式执行：`rustdoc --test`.  实例：

    `./x.py test src/doc`-运行`rustdoc --test`所有文件`src/doc`.

    `./x.py test --doc src/libstd`-运行`rustdoc --test`在标准库中。

-   **链接检查器**–用于验证的小工具`href`文档中的链接。

    例子：`./x.py test src/tools/linkchecker`

-   **点检**–这将验证由构建系统创建的源分发tarball将解包、构建和运行所有测试。

    例子：`./x.py test distcheck`

-   **工具试验**–含铁锈的包装也进行了所有测试（通常通过运行`cargo test`在他们的目录中）。这包括货物、clippy、rustfmt、rls、miri、bootstrap（测试rust构建系统本身）等。

-   **货物检验**–这是一个运行的小工具`cargo test`一些重要的项目（如`servo`，`ripgrep`，`tokei`以确保没有任何显著的回归。

    例子：`./x.py test src/tools/cargotest`

## 测试基础设施

当在GitHub上打开拉请求时，[特拉维斯]将自动启动将在单个配置（x86-64 Linux）上运行所有测试的生成。本质上，它运行`./x.py test`建成后。

集成机器人[博尔斯]用于协调到主分支的合并。当一个pr被批准时，它将进入[队列]在使用Travis和[追随者]（目前超过50种不同配置）。大多数平台只运行构建步骤，有些只运行一组受限的测试，只有一个子集运行完整的测试套件（参见Rust的[平台层]）

[travis]: https://travis-ci.org/rust-lang/rust

[bors]: https://github.com/servo/homu

[queue]: https://buildbot2.rust-lang.org/homu/queue/rust

[appveyor]: https://ci.appveyor.com/project/rust-lang/rust

[platform tiers]: https://forge.rust-lang.org/platform-support.html

## 用Docker图像测试

铁锈树包括[码头工人]Travis-In上使用的平台的图像定义[src/ci/docker].  剧本[源代码/ci/docker/run.sh]用于构建Docker映像、运行它、在映像中构建信任并运行测试。

> TODO:在您不容易访问的平台上测试/调试的典型工作流是什么？人们会建立Docker图像并输入它们来测试吗？

[docker]: https://www.docker.com/

[src/ci/docker]: https://github.com/rust-lang/rust/tree/master/src/ci/docker

[src/ci/docker/run.sh]: https://github.com/rust-lang/rust/blob/master/src/ci/docker/run.sh

## 在模拟器上测试

有些平台是通过模拟器来测试那些不容易获得的体系结构的。有一组工具用于协调在模拟器中运行测试。平台，如`arm-android`和`arm-unknown-linux-gnueabihf`设置为在Travis上自动运行模拟测试。下面将介绍目标的测试是如何在模拟下运行的。

码头工人形象[ARMHF GNU-]包括[齐母]模拟ARM CPU体系结构。铁锈树里有工具[远程测试客户端-]和[远程测试服务器-]这些程序用于将测试程序和库发送到仿真器，在仿真器中运行测试，并读取结果。Docker图像已设置为启动`remote-test-server`构建工具使用`remote-test-client`与服务器通信以协调运行的测试（请参见[SRC/引导程序/测试.rs]）

> TODO:在模拟器中手动运行测试的步骤是什么？`./src/ci/docker/run.sh armhf-gnu`将做所有的事情，但运行需要数小时，并且在模拟器内交互没有提供太多帮助。
>
> 是否支持模拟其他（非Android）平台，如在iOS模拟器上运行？
>
> 关于在真正的硬件上远程运行测试，这里还有什么有趣的地方可以说吗？
>
> 我也不清楚WASM或ASM.JS测试是如何运行的。

[armhf-gnu]: https://github.com/rust-lang/rust/tree/master/src/ci/docker/armhf-gnu

[qemu]: https://www.qemu.org/

[remote-test-client]: https://github.com/rust-lang/rust/tree/master/src/tools/remote-test-client

[remote-test-server]: https://github.com/rust-lang/rust/tree/master/src/tools/remote-test-server

[src/bootstrap/test.rs]: https://github.com/rust-lang/rust/tree/master/src/bootstrap/test.rs

## 陨石坑

[陨石坑](https://github.com/rust-lang-nursery/crater)是编译和运行测试的工具*每一个*板条箱[克拉西奥](https://crates.io)（还有一些在Github上）。它主要用于在实现可能中断的更改时检查中断程度，并通过运行beta与稳定编译器版本来确保没有中断。

### 何时运行火山口

如果您的pr对编译器进行了较大的更改或者可能导致崩溃，那么您应该请求运行一个坑。如果你不确定，可以问问你的公关评论员。

### 请求弹坑运行

Rust团队维护了一些机器，这些机器可用于运行PR引入的更改后的环形山运行。如果您的PR需要运行环形山，请在PR线程中为Triage团队留下注释。请告知团队您是否需要“仅检查”环形山运行、“仅构建”环形山运行或“构建和测试”环形山运行。区别主要在于时间；保守的（如果您不确定）选项是进行构建和测试运行。如果所做的更改只会在编译时产生效果（例如，实现一个新特性），那么您只需要一次检查运行。

您的公关将由分诊团队排队，结果将在准备好后公布。检查运行大约需要3-4天，另外两个平均需要5-6天。

虽然陨石坑真的很有用，但也要注意一些注意事项：

-   并非所有代码都在板条箱上。在Github和其他地方的repos中有很多代码。此外，公司可能不希望发布其代码。因此，一次成功的陨石坑运行并不是一个神奇的绿灯，不会有破碎；你仍然需要小心。

-   Rample只运行基于x86\_的Linux版本。因此，其他架构和平台没有经过测试。关键是，这包括窗口。

-   许多板条箱没有经过测试。这可能有很多原因，包括板条箱不再编译（例如，使用的旧的夜间功能），有损坏或剥落的测试，需要网络访问，或其他原因。

-   在火山口运行之前，`@bors try`需要成功地构建工件。这意味着如果你的代码不编译，你就不能运行火山口。

## PARF运行

在提高编译器的性能和防止性能回归方面做了大量的工作。“perf run”用于比较编译器在不同配置下对大量流行板条箱的性能。不同的配置包括“新构建”、带有增量编译的构建等。

性能运行的结果是编译器的两个版本之间的比较（通过它们的提交哈希）。

如果您的pr可能会影响性能，特别是如果它可能会对性能产生不利影响，则应请求执行perf run。

## 进一步阅读

以下博客文章可能也很有趣：

-   布森经典[“如何测试生锈”][howtest]

[howtest]: https://brson.github.io/2017/07/10/how-rust-is-tested
