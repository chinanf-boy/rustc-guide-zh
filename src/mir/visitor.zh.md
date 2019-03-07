# 米尔游客

mir访问者是一个方便的工具，可以遍历mir，或者查找事物，或者对其进行更改。访客特征的定义见[这个`rustc::mir::visit`模块][m-v]–其中有两个是通过单个宏生成的：`Visitor`（在`&Mir`并返回共享的引用）和`MutVisitor`（在`&mut Mir`并返回可变引用）。

[m-v]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/mir/visit/index.html

要实现访问者，必须创建表示访问者的类型。通常，这种类型希望在处理mir时“挂起”到您需要的任何状态：

```rust,ignore
struct MyVisitor<...> {
    tcx: TyCtxt<'cx, 'tcx, 'tcx>,
    ...
}
```

然后执行`Visitor`或`MutVisitor`这种类型的特征：

```rust,ignore
impl<'tcx> MutVisitor<'tcx> for NoLandingPads {
    fn visit_foo(&mut self, ...) {
        ...
        self.super_foo(...);
    }
}
```

如上图所示，在IMPL中，您可以覆盖`visit_foo`方法（例如，`visit_terminator`）为了编写一些代码，每当`foo`被发现。如果要递归地遍历`foo`，然后调用`super_foo`方法。（NB）。你永远不想覆盖`super_foo`）

一个非常简单的访客例子可以在[`NoLandingPads`]. 该访问者甚至不需要任何状态：它只访问所有终止符并删除它们`unwind`接班人。

[`nolandingpads`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/no_landing_pads/struct.NoLandingPads.html

## 遍历

除了访客，[这个`rustc::mir::traversal`模块][t]包含在mir cfg中执行的有用函数[不同的标准订单][traversal]（例如预订单、反向订单等）。

[t]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/mir/traversal/index.html

[traversal]: https://en.wikipedia.org/wiki/Tree_traversal
