# 更新LLVM

Rust编译器今天使用LLVM作为其主要的codegen后端，当然我们希望至少偶尔更新这种依赖！目前，我们没有关于何时更新LLVM或可以更新的更严格的策略，但是应用了一些指导原则：

-   我们尝试始终支持最新发布的LLVM版本
-   我们尝试支持LLVM的“最后几个”版本（有多少版本随时间变化）
-   我们允许在开发期间转移到任意提交。
-   在将其包含在rustc中之前，强烈倾向于将所有修补程序上游到LLVM。

这个政策可能会随着时间的推移而改变（或者实际上可能会作为正式的政策开始存在！），但是现在这些都是粗略的指导方针！

## 为何更新LLVM？

现在有两个主要原因我们想以某种方式更新LLVM：

-   首先，一个错误可以修复！我们经常在编译器中发现错误并将它们固定在LLVM的上游。我们希望将修复程序恢复到编译器本身，因为它们已合并到上游。

-   其次，我们希望在rustc中使用LLVM中的新功能，但我们不想等待完整的LLVM版本来测试它。

这些原因中的每一个都有不同的更新LLVM的策略，我们将在这里详细介绍。

## 错误修正

对于通常只更新错误的LLVM的更新，我们挑选错误修复到我们已经使用的分支。这个步骤是：

1.  确保错误修复位于上游LLVM中。
2.  确定rustc当前正在使用的分支。该`src/llvm`子模块总是固定在一个分支上[锈琅/ LLVM](https://github.com/rust-lang/llvm)库。
3.  分叉rust-lang / llvm存储库
4.  查看相应的分支（通常命名为`rust-llvm-release-*`）
5.  樱桃 - 选择上游提交到分支
6.  将此分支推入您的分支
7.  向之前的同一分支发送一个pull-lang / llvm的Pull请求
8.  等待PR合并
9.  发送PR到rust-lang / rust更新`src/llvm`子模块与您的错误修复
10. 等待PR合并

tl; dr;是我们可以随时挑选错误修正并将它们拉回到我们正在使用的rust-lang / llvm分支中，并将它放入编译器只是通过PR更新子模块！

示例PR看起来像：[＃56313](https://github.com/rust-lang/rust/pull/56313)

## 功能更新

> 请注意，这是适用于当前年龄的所有信息。几乎所有LLVM更新都会更新LLVM更改的过程，因此这可能已过时！

与错误修正不同，更新以获取LLVM的新功能通常需要更多工作。这是我们无法合理地向后提交提交的地方，因此我们需要进行全面更新。这里有很多东西要做，所以让我们详细介绍一下。

1.  在此更新的所有存储库中创建新分支。应该命名分支`rust-llvm-release-X-Y-Z-vA`哪里`X.Y.Z`是LLVM版本和`A`只是基于此名称的先前分支是否正在增加。这里的所有存储库都应该从我们目前使用的上游LLVM项目中同时分支<https://github.com/llvm-mirror>库。需要新分支的存储库列表是：

    -   锈琅/ LLVM
    -   锈琅/编译-RT
    -   锈琅/ LLD
    -   锈琅苗圃/ LLDB
    -   锈琅苗圃/铛

2.  将特定于Rust的修补程序应用于LLVM存储库。所有功能和错误修正都在上游，但通常会有一些奇怪的与构建相关的补丁对我们的存储库上的上游没有意义。这些补丁通常是分支上的最新补丁。所有存储库，除外`clang`，目前有特定于Rust的补丁。

3.  更新`compiler-rt`子模块`rust-lang-nursery/compiler-builtins`库。将此更新推送到`rust-llvm-release-*`的分支`compiler-builtins`库。

4.  准备生锈/生锈

-   更新`src/llvm`
-   更新`src/tools/lld`
-   更新`src/tools/lldb`
-   更新`src/tools/clang`
-   更新\`src / libcompiler_builtins
-   编辑`src/rustllvm/llvm-rebuild-trigger`更新其内容

5.  建立你的提交。确保您已提交先前的更改，以确保不还原子模块更新。您应该执行的一些命令是：

    -   `./x.py build src/llvm`- 测试LLVM是否仍然构建
    -   `./x.py build src/tools/lld`- 同样适用于LLD
    -   `./x.py build`- 构建其余的rustc

    您可能需要更新`src/rustllvm/*.cpp`使用更新的LLVM绑定进行编译。请注意，您应该使用`#ifdef`这样可以确保绑定仍然可以在较旧的LLVM版本上编译。

6.  测试其他平台的回归。LLVM通常至少有一个针对非第1层架构的错误，因此在将其发送给bors之前做一些更多的测试是很好的！如果你的资源很少，你现在就可以将PR发送给bors，并且无论如何它都会被测试。

    理想情况下，构建LLVM并在几个平台上测试它：

    -   Linux的
    -   OSX
    -   视窗

    然后运行CI也做的一些docker容器：

    -   `./src/ci/docker/run.sh wasm32-unknown`
    -   `./src/ci/docker/run.sh arm-android`
    -   `./src/ci/docker/run.sh dist-various-1`
    -   `./src/ci/docker/run.sh dist-various-2`
    -   `./src/ci/docker/run.sh armhf-gnu`

7.  发送公关！希望从这里顺利航行:)。

对于现有技术，先前的LLVM更新看起来像[＃55835](https://github.com/rust-lang/rust/pull/55835)
[＃47828](https://github.com/rust-lang/rust/pull/47828)

### 警告和陷阱

理想情况下，上面的说明非常顺利，但在通过它们时需要记住一些注意事项：

-   LLVM漏洞很难找到，请不要犹豫，寻求帮助！Bisection绝对是你的朋友（是的LLVM需要永远建立，但是bisection仍然是你的朋友）
-   更新LLDB目前有一些特定于Rust的补丁不在上游。如果你有困难@tromey可能会有所帮助。
-   如果您有一般性问题，@ alexcrichton可以帮助您。
-   创建分支是GitHub上的特权操作，因此您需要具有写访问权限的人来为您创建分支。
