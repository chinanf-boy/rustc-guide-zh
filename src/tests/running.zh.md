# 运行测试

您可以使用`x.py`运行测试。最基本的命令 —— 也是最不想使用的！–如下：

```bash
> ./x.py test
```

这将构建完整的阶段 2 编译器，然后运行整个测试套件。你可能不想经常这样做，因为这需要很长的时间，而且不管怎样，bors/Travis 会为你这样做。（通常，我会在打开一个我认为已经完成的 pr 之后，在后台运行这个命令，但很少这样做。-nmatsakis）

测试结果会缓存起来，以前，测试期间成功的测试会`ignored(忽略)`。stdout/stderr 的内容以及每个测试的时间戳文件，可以在`build/ARCH/test/`下找到。要强制重新运行测试（例如，如果测试运行程序没有注意到更改，可以简单删除时间戳文件。

注意，一些测试需要 Python-启用 的 gdb。您可以使用 gdb 的`python`命令，看看 你的 gdb 支不支持 Python。一旦能调用后，可以键入一些 python 代码（例如`print("hi")`），然后`CTRL+D`执行它。如果要从源代码构建 gdb，则需要使用配置`--with-python=<path-to-python-binary>`。

## 运行测试套件的子集

在处理特定的 pr 时，您通常希望运行一组较小的测试，并使用阶段 1 编译器。例如，在修改 rustc 后，可以使用一个好的“烟雾测试”，以查看事情是否正常工作，如下所示：

```bash
> ./x.py test --stage 1 src/test/{ui,compile-fail,run-pass}
```

这将运行`ui`，`compile-fail`和`run-pass`测试套件，和只有第一阶段的构建。当然，测试套件的选择有些随意，可能不适合您正在执行的任务。例如，如果您正在对 debuginfo 进行 hacking，那么最好使用 debuginfo 测试套件：

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

### 使用阶段 1 编译器在标准库上运行测试

```bash
>   ./x.py test src/libstd --stage 1
```

通过列出要运行的测试套件，可以避免对根本没有更改的组件运行测试。

**警告：**请注意，bors 只在完整的第 2 阶段构建中运行测试；因此，当测试**常**在第 1 阶段工作正常，要知道有一些限制。尤其是，Stage1 编译器不能很好地处理程序宏或自定义派生的测试。

## 运行单个测试

人们想做的另一件事是**单个测试**，通常是他们试图修复的测试。一种方法是调用`x.py`与`--test-args`选项：

```bash
> ./x.py test --stage 1 src/test/ui --test-args issue-1234
```

在引擎盖下，测试运行程序调用标准的 Rust 测试运行程序（与您使用的`#[test]`相同），所以这个命令会筛选名称中，包含“issue-1234”的测试。

## 使用增量编译

您可以进一步启用`--incremental`标志，用于在后续重建中，节省额外时间：

```bash
> ./x.py test --stage 1 src/test/ui --incremental --test-args issue-1234
```

如果不想在每个命令中包含标志，也可以在`config.toml`：

```toml
# Whether to always use incremental compilation when building rustc
incremental = true
```

请注意，增量编译将比通常使用更多的磁盘空间。如果您担心磁盘空间问题，时刻记得检查`build`目录的大小。

## 手动运行测试

有时候，手动进行测试会更容易、更快，因大多数测试只是`rs`文件，那么你就可以做

```bash
> rustc +stage1 src/test/ui/issue-1234.rs
```

这速度快得多，但并不总是有效。例如，一些测试包括，特定编译器标志的指令，或者依赖于其他箱的指令，如果没有这些选项，它们可能无法运行相同的指令。
