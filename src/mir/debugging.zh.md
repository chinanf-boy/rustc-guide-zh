# miR调试

这个`-Zdump-mir`标记可用于转储mir的文本表示形式。这个`-Zdump-mir-graphviz`标志可用于转储`.dot`将mir表示为控制流图的文件。

`-Zdump-mir=F`是一个方便的编译器选项，可以让您在编译的每个阶段查看每个函数的mir。`-Zdump-mir`采取了**滤波器** `F`它允许您控制您感兴趣的函数和传递。例如：

```bash
> rustc -Zdump-mir=foo ...
```

这将为名称包含的任何函数转储mir`foo`每次通过之前和之后，它都会丢弃MIR。这些文件将在`mir_dump`目录。可能会有很多这样的人！

```bash
> cat > foo.rs
fn main() {
    println!("Hello, world!");
}
^D
> rustc -Zdump-mir=main foo.rs
> ls mir_dump/* | wc -l
     161
```

这些文件的名称如下`rustc.main.000-000.CleanEndRegions.after.mir`. 这些名称包含多个部分：

```text
rustc.main.000-000.CleanEndRegions.after.mir
      ---- --- --- --------------- ----- either before or after
      |    |   |   name of the pass
      |    |   index of dump within the pass (usually 0, but some passes dump intermediate states)
      |    index of the pass
      def-path to the function etc being dumped
```

你也可以做更多的选择性过滤器。例如，`main & CleanEndRegions`将选择引用的内容*二者都* `main`和通行证`CleanEndRegions`：

```bash
> rustc -Zdump-mir='main & CleanEndRegions' foo.rs
> ls mir_dump
rustc.main.000-000.CleanEndRegions.after.mir	rustc.main.000-000.CleanEndRegions.before.mir
```

过滤器也可以`|`要组合多组的部件`&`-过滤器。例如`main & CleanEndRegions | main &
NoLandingPads`将选择*任何一个* `main`和`CleanEndRegions` *或*
`main`和`NoLandingPads`：

```bash
> rustc -Zdump-mir='main & CleanEndRegions | main & NoLandingPads' foo.rs
> ls mir_dump
rustc.main-promoted[0].002-000.NoLandingPads.after.mir
rustc.main-promoted[0].002-000.NoLandingPads.before.mir
rustc.main-promoted[0].002-006.NoLandingPads.after.mir
rustc.main-promoted[0].002-006.NoLandingPads.before.mir
rustc.main-promoted[1].002-000.NoLandingPads.after.mir
rustc.main-promoted[1].002-000.NoLandingPads.before.mir
rustc.main-promoted[1].002-006.NoLandingPads.after.mir
rustc.main-promoted[1].002-006.NoLandingPads.before.mir
rustc.main.000-000.CleanEndRegions.after.mir
rustc.main.000-000.CleanEndRegions.before.mir
rustc.main.002-000.NoLandingPads.after.mir
rustc.main.002-000.NoLandingPads.before.mir
rustc.main.002-006.NoLandingPads.after.mir
rustc.main.002-006.NoLandingPads.before.mir
```

（这里，`main-promoted[0]`文件引用mir中出现的“提升常量”`main`函数。

托多：还有别的吗？
