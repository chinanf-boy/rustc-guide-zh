# 运行试验

您可以使用`x.py`. 最基本的命令——你几乎不想使用它！–如下：

```bash
> ./x.py test
```

这将构建完整的阶段2编译器，然后运行整个测试套件。你可能不想经常这样做，因为这需要很长的时间，而且不管怎样，Bors/Travis会为你这样做。（通常，我会在打开一个我认为已经完成的pr之后在后台运行这个命令，但很少这样做。- NMASSAKIS

将缓存测试结果，以前成功的测试`ignored`测试期间。stdout/stderr内容以及每个测试的时间戳文件可以在下找到`build/ARCH/test/`. 要强制重新运行测试（例如，如果测试运行程序没有注意到更改），您可以简单地删除时间戳文件。

注意，一些测试需要启用python的gdb。您可以使用`python`来自gdb的命令。调用后，可以键入一些python代码（例如`print("hi")`）然后返回，然后`CTRL+D`执行它。如果要从源代码构建gdb，则需要使用配置`--with-python=<path-to-python-binary>`.

## 运行测试套件的子集

在处理特定的pr时，您通常希望运行一组较小的测试，并使用阶段1构建。例如，在修改rustc后，可以使用一个好的“烟雾测试”，以查看事情是否正常工作，如下所示：

```bash
> ./x.py test --stage 1 src/test/{ui,compile-fail,run-pass}
```

这将运行`ui`，`compile-fail`和`run-pass`测试套件，只有第一阶段的构建。当然，测试套件的选择有些随意，可能不适合您正在执行的任务。例如，如果您正在对debuginfo进行黑客攻击，那么最好使用debuginfo测试套件：

```bash
> ./x.py test --stage 1 src/test/debuginfo
```

### 只运行整洁的脚本

```bash
> ./x.py test src/tools/tidy
```

### 在标准库上运行测试

```bash
> ./x.py test src/libstd
```

### 在标准库上运行测试并运行整洁的脚本

```bash
> ./x.py test src/libstd src/tools/tidy
```

### 使用阶段1编译器在标准库上运行测试

```bash
>   ./x.py test src/libstd --stage 1
```

通过列出要运行的测试套件，可以避免对根本没有更改的组件运行测试。

**警告：**请注意，BOR只在完整的第2阶段构建中运行测试；因此，当测试**通常**第1阶段工作正常，有一些限制。尤其是，Stage1编译器不能很好地处理程序宏或自定义派生测试。

## 运行单个测试

人们想做的另一件事是**个别测试**，通常是他们试图修复的测试。一种方法是调用`x.py`与`--test-args`选项：

```bash
> ./x.py test --stage 1 src/test/ui --test-args issue-1234
```

在引擎盖下，测试运行程序调用标准的Rust测试运行程序（与您使用的测试运行程序相同）`#[test]`，所以这个命令将结束对名称中包含“issue-1234”的测试的筛选。

## 使用增量编译

您可以进一步启用`--incremental`用于在后续重建中节省额外时间的标志：

```bash
> ./x.py test --stage 1 src/test/ui --incremental --test-args issue-1234
```

如果不想在每个命令中包含标志，可以在`config.toml`也：

```toml
# Whether to always use incremental compilation when building rustc
incremental = true
```

请注意，增量编译将比通常使用更多的磁盘空间。如果您担心磁盘空间问题，您可能需要检查`build`目录。

## 手动运行测试

有时候用手进行测试会更容易、更快。大多数测试只是`rs`文件，这样你就可以做

```bash
> rustc +stage1 src/test/ui/issue-1234.rs
```

这速度快得多，但并不总是有效。例如，一些测试包括指定特定编译器标志的指令，或者依赖于其他板条箱的指令，如果没有这些选项，它们可能无法运行相同的指令。
