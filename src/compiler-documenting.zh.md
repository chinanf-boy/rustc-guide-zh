# 记录rustc

您可能希望构建可用的各种组件的文档，如标准库。有两种方法可以解决这个问题。您可以直接在文件上运行rustdoc以确保HTML正确，这很快。或者，您可以通过x.py在构建过程中构建文档。两者都是可行的方法，因为文档更多地是关于内容。

## 记录一切

```bash
./x.py doc
```

## 如果你想避免整个Stage 2构建

```bash
./x.py doc --stage 1
```

首先构建编译器和rustdoc以确保一切正常，然后记录文件。

## 记录特定组件

```bash
   ./x.py doc src/doc/book
   ./x.py doc src/doc/nomicon
   ./x.py doc src/doc/book src/libstd
```

与单个测试或构建某些组件非常相似，您只能构建所需的文档。

## 记录内部防锈物品

默认情况下不构建编译器文档。config.toml中有一个用于实现相同目标的标志。但是，启用后，编译器文档确实包含内部项。

接下来打开config.toml并确保将这两行设置为true：

```bash
docs = true
compiler-docs = true
```

如果要构建编译器文档，请运行以下命令：

```bash
./x.py doc
```

这将看到docs和compiler-docs选项设置为true并构建通常隐藏的编译器文档！

### 编译器文档

有关防锈组件的文档可在以下位置找到：[生锈的医生]。

[rustc doc]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/
