# 文档化 rustc

您可能希望构建可用的各种组件的文档，如标准库。有两种方法可以解决这个问题。您可以直接在文件上运行 rustdoc ，以确保 HTML 正确，这很快。或者，您可以通过 x.py 在构建过程中，构建文档。两者都是可行的方法，鉴于文档只是更多内容。

## 文档化一切

```bash
./x.py doc
```

## 如果你想避免整个 Stage 2 构建

```bash
./x.py doc --stage 1
```

首先构建编译器和 rustdoc ，以确保一切正常，然后文档化文件。

## 文档化特定组件

```bash
   ./x.py doc src/doc/book
   ./x.py doc src/doc/nomicon
   ./x.py doc src/doc/book src/libstd
```

与单个测试或构建某些组件非常相似，您只能构建所需的文档。

## 文档化 rustc 内部项

默认情况下，不构建编译器文档。config.toml 中有一个用于实现相同目标的标志。但是，启用后，编译器文档确实包含内部项。

接下来打开 config.toml 并确保将这两行设置为 true：

```bash
docs = true
compiler-docs = true
```

如果要构建编译器文档，请运行以下命令：

```bash
./x.py doc
```

可以看到 docs 和 compiler-docs 选项都设置为 true ，那么，这会构建通常隐藏了的编译器文档！

### 编译器文档

有关 rust 组件的文档，可在以下位置找到：[rustc doc]。

[rustc doc]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/
