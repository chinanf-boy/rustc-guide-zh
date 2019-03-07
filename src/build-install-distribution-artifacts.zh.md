# 构建分布工件

您可能希望构建并打包编译器以进行分发。您将要运行此命令来执行此操作：

```bash
./x.py dist
```

# 安装分发工件

如果您构建了分发工件，则可能需要安装它并测试它是否适用于目标系统。您将要运行此命令：

```bash
./x.py install
```

注意：如果要测试对编译器的修改，可能需要使用它来编译某个项目。通常，您不希望使用./x.py install进行测试。相反，你应该创建一个工具链，如中所述[这里][create-rustup-toolchain]。

例如，如果您创建的工具链被称为foo，那么您将使用它来调用它`rustc +foo ...`（其中......表示其余参数）。

[create-rustup-toolchain]: ./how-to-build-and-run.md#creating-a-rustup-toolchain
