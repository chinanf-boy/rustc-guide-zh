# 编译器的测试框架

Rust 项目运行各种不同的测试，由构建系统协调（`x.py test`）。编译器本身的主要测试工具是一个名为 compiletest 的工具（源代码位于[`src/tools/compiletest`]）本节简要概述了如何设置测试框架，然后详细介绍了[如何运行测试](./running.zh.html#ui)以及[如何添加新测试](./adding.zh.html).

[`src/tools/compiletest`]: https://github.com/rust-lang/rust/tree/master/src/tools/compiletest

## 编译器的测试套件

编译器的测试位于[`src/test`]目录。您将立即看到一系列子目录（例如`ui`，`run-make`等等）。每个目录都称为一个**测试套件** —— 它们包含一组以不同模式运行的测试。

[`src/test`]: https://github.com/rust-lang/rust/tree/master/src/test

以下是本文中，测试套件的简要概述以及它们的含义。在某些情况下，测试套件链接到手册中，会提供更多详细信息的部分。

- [`ui`](./adding.zh.html#ui) —— 检查编译和/或测试运行时的准确 stdout/stderr 的测试
- `run-pass` —— 预期编译和执行成功的测试（无恐慌）
  - `run-pass-valgrind` —— 应使用 Valgrind 进行的测试
- `run-fail` —— 编译的测试，预期在执行过程中会死机
- `compile-fail`—预期编译失败的测试。
- `parse-fail` —— 预期无法解析的测试
- `pretty` —— 针对 Rust“pretty 打印”的测试，它从 AST 生成有效的 Rust 代码。
- `debuginfo` —— 在 gdb 或 lldb 中运行，并查询调试信息的测试
- `codegen` —— 编译测试，去测试生成的 LLVM 代码，以确保我们想要的优化正在生效。
- `mir-opt` —— 测试检查生成的 MIR 的各个部分，以确保正确的构建内容或执行预期的优化。
- `incremental` —— 测试增量编译，检查在执行某些修改时，我们是否能够重用以前编译的结果。
- `run-make` —— 基本上只执行`Makefile`; 已是灵活性的极限，但写起来很烦人。
- `rustdoc` —— 对 rustdoc 进行测试，确保生成的文件包含预期的文档。
- `*-fulldeps` —— 同上，但该测试表明，依赖除`libstd`之外的东西（因此这些东西必须建造）

## 其他测试

Rust Build 系统可处理各种其他事项的运行测试，包括：

- **整洁(Tidy)** —— 这是一个自定义工具，用于验证源代码样式和格式风格，例如拒绝长行。有更多的信息在[关于编码风格的章节](../conventions.zh.html#formatting).

  例：`./x.py test src/tools/tidy`

- **单元测试** —— Rust 标准库和许多 Rust 包，包括典型的 Rust`#[test]`单元测试。在引擎盖下，`x.py`将在每个包上启动`cargo test`，运行所有测试。

  例：`./x.py test src/libstd`

- **文档测试** —— Rust 文档中嵌入的示例代码，通过以下方式执行：`rustdoc --test`。 实例：

  `./x.py test src/doc` —— 为`src/doc`中的所有文件运行`rustdoc --test`。

  `./x.py test --doc src/libstd` —— 在标准库中运行`rustdoc --test`。

- **链接检查器** —— 小工具，用于验证文档中的`href`链接。

  例：`./x.py test src/tools/linkchecker`

- **分发 检查** —— 这将验证由构建系统伙同源代码，创建的分发压缩文件，对其解包、构建和运行所有测试。

  例：`./x.py test distcheck`

- **工具试验** —— 含 Rust 在内的包也要进行了所有测试（通常在他们的目录中，运行`cargo test`）。这包括 Cargo、clippy、rustfmt、rls、miri、bootstrap（测试 Rust 构建系统本身）等。

- **Cargo 检验** —— 这是一个小工具，在一些重要的项目上，运行`cargo test`（如`servo`，`ripgrep`，`tokei`以确保没有任何显著的性能退步。

  例：`./x.py test src/tools/cargotest`

## 测试基础设施

当在 GitHub 上打开 PR 时，[Travis]自动启动构建，将在单个配置（x86-64 Linux）上运行所有测试。本质上，它在构建后运行`./x.py test`。

集成机器人[bors]用于协调到主分支的合并。当一个 pr 被批准时，它将进入[queue]列表。该列表被 Travis 和[Appveyor]使用，会在一组广泛的平台上，一次测试一个合并（目前超过 50 种不同配置）。大多数平台只运行构建步骤，有些只运行一组有限的，只有一个子集运行完整的测试套件（参见 Rust 的[平台层][platform tiers]）

[travis]: https://travis-ci.org/rust-lang/rust
[bors]: https://github.com/servo/homu
[queue]: https://buildbot2.rust-lang.org/homu/queue/rust
[appveyor]: https://ci.appveyor.com/project/rust-lang/rust
[platform tiers]: https://forge.rust-lang.org/platform-support.html

## 用 Docker 镜像测试

Rust 树包括，在 Travis 上使用[src/ci/docker]的平台[Docker]镜像。 该脚本[src/ci/docker/run.sh]用于构建 Docker 镜像、运行它、在镜像中构建 Rust 并运行测试。

> TODO:在您不容易访问的平台上，测试/调试的典型工作流是什么？人们会构建 Docker 镜像，并进到里面测试吗？

[docker]: https://www.docker.com/
[src/ci/docker]: https://github.com/rust-lang/rust/tree/master/src/ci/docker
[src/ci/docker/run.sh]: https://github.com/rust-lang/rust/blob/master/src/ci/docker/run.sh

## 在模拟器上测试

有些平台是那些不常用的体系结构，所以通过模拟器来测试。有一组工具协调在模拟器的测试运行。如`arm-android`和`arm-unknown-linux-gnueabihf`平台设置为在 Travis 上自动运行模拟器测试。下面将介绍目标的测试是，如何在模拟器下运行的。

[armhf-gnu]的 Docker 镜像，会包括[QEMU]，它能模拟 ARM CPU 体系结构。Rust 树里有[remote-test-client]和[remote-test-server]工具。这些程序把测试程序和库发送到模拟器，在模拟器中运行测试，并读取结果。Docker 镜像已设为启动`remote-test-server`服务器，和构建工具使用`remote-test-client`与服务器通信，以协调运行的测试（请参见[src/bootstrap/test.rs]）

> TODO:在模拟器中手动运行测试的步骤是什么？
> 虽说`./src/ci/docker/run.sh armhf-gnu`会做所有的事情，但运行需要数小时，且在与模拟器交互上没有提供太多帮助。
>
> 是否支持模拟其他（非 Android）平台，如在 iOS 模拟器上运行？
>
> 关于在真正的硬件上远程运行测试，这里还有什么有趣的地方可以说吗？
>
> 我也不清楚 wasm 或 asm.js 测试是如何运行的。

[armhf-gnu]: https://github.com/rust-lang/rust/tree/master/src/ci/docker/armhf-gnu
[qemu]: https://www.qemu.org/
[remote-test-client]: https://github.com/rust-lang/rust/tree/master/src/tools/remote-test-client
[remote-test-server]: https://github.com/rust-lang/rust/tree/master/src/tools/remote-test-server
[src/bootstrap/test.rs]: https://github.com/rust-lang/rust/tree/master/src/bootstrap/test.rs

## Crater

[Crater](https://github.com/rust-lang-nursery/crater)是编译和运行测试的工具，服务[crates.io](https://crates.io)（还有一些在 Github 上） 上的 *每一个*箱。它主要用于在实现潜在的破坏性 API 更改时，检查破损程度，并通过运行 beta 与稳定编译器版本的比较，确保没有。

### 何时运行 Crater

如果您的 pr 对编译器进行了较大的更改或者可能导致崩溃，那么您应该请求运行一个 Crater。如果你不确定，可以问问你的 PR 审查员。

### 请求 Crater 运行

Rust 团队维护了一些机器，这些机器可用于 crater 的运行，求证 PR 引入的更改 。如果您的 PR 需要运行 crater，请在该 PR 留言版中为 Triage 团队留下讨论。请告知团队您需要的，是否为一个“仅检查”crater 运行、还是一个“仅构建”crater 运行或“构建和测试”crater 的运行。区别主要在于时间；保守的（如果您不确定）选项是进行“构建和测试”运行。如果所做的更改只会在编译时产生效果（例如，实现一个新 trait），那么您只需要一次“仅检查”运行。

您的 PR 将由 Triage 团队分配，结果在准备好后公布。“仅检查”运行大约需要 3-4 天，另外两个平均需要 5-6 天。

虽然 Crater 真的很有用，但也要注意一些注意事项：

- 并非所有代码都在 crates.io 上。在 Github 和其他地方的 repos 中也有很多代码。此外，某些公司可能不希望发布他们代码。因此，一次成功的 Crater 运行并不是一个神奇的，不会有破碎的绿灯；你仍然需要小心。

- Crater 只运行在基于 x86_64 的 Linux 版本。因此，其他架构和平台没有经过测试。当然，包括 Windows。

- 许多箱没有经过测试，这可能有很多原因，包括箱不再能编译（例如，使用的旧的 nightly 特性），有损坏或剥落的测试，需要网络访问，或其他原因。

- 在 Crater 运行之前，`@bors try`需要在构建工件成功。这意味着如果你的代码不能编译，你就不能运行 Crater。

## 性能运行

团队在提高编译器的性能，和防止性能退步方面做了大量的工作。“perf run”用于比较编译器在不同配置下，对大量流行箱的性能。不同的配置包括“新的构建”、带有增量编译的构建等。

性能(pref)运行的结果是编译器的两个版本之间的比较（通过它们的提交哈希）。

如果您的 pr 可能会影响性能，特别是如果它可能会对性能产生不利影响，则应请求执行 perf run。

## 进一步阅读

以下博客文章，可能也很有趣：

- 布森的经典，[“ Rust 如何测试”][howtest]

[howtest]: https://brson.github.io/2017/07/10/how-rust-is-tested
