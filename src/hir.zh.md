# HIR

HIR——“高级中间表示法”——是大多数RustC中使用的主要IR。它是在解析、宏扩展和名称解析之后生成的抽象语法树（ast）的编译器友好表示（请参见[降低](./lowering.html)关于如何创建HIR）。hir的许多部分与rust surface语法非常相似，只是rust的一些表达形式已经被删除了。例如，`for`循环转换为`loop`不要出现在HIR中。这使得hir比正常的ast更易于分析。

本章介绍了HIR的主要概念。

您可以通过传递`-Zunpretty=hir-tree`RUSTC旗：

```bash
> cargo rustc -- -Zunpretty=hir-tree
```

### 带外存储和`Crate`类型

HIR中的顶级数据结构是[`Crate`]它存储当前正在编译的板条箱的内容（我们只为当前板条箱构建HIR）。而在AST中，板条箱数据结构基本上只包含根模块hir`Crate`结构包含许多地图和其他东西，用于组织板条箱的内容，以便于访问。

[`crate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Crate.html

例如，HIR中单个项目的内容（例如模块、功能、特征、impl等）在父级中不能立即访问。例如，如果存在模块项`foo`包含函数`bar()`：

```rust
mod foo {
    fn bar() { }
}
```

然后在HIR中，模块的表示`foo` (the [`Mod`]结构）将只具有**`ItemId`** `I`属于`bar()`. 获取函数的详细信息`bar()`，我们会查找`I`在`items`地图。

[`mod`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Mod.html

这种表示的一个很好的结果是，通过迭代这些映射中的键值对，可以迭代板条箱中的所有项（不需要拖拽整个hir）。对于特征项和impl项，以及“body”（解释如下）都有类似的映射。

以这种方式设置表示的另一个原因是为了更好地与增量编译集成。这样，如果您可以访问[`&hir::Item`]（例如，对于国防部`foo`）不会立即访问函数的内容。`bar()`. 相反，您只能访问**身份证件**对于`bar()`，并且必须调用一些函数来查找`bar()`给定其ID；这使编译器有机会观察您访问的数据`bar()`，然后记录依赖项。

[`&hir::item`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Item.html

<a name="hir-id"></a>

### HIR中的标识符

大多数必须在hir中处理事物的代码倾向于不将引用带到hir中，而是带到hir中。*标识符编号*（或者只是“ID”）。现在，您将发现四种正在使用的标识符：

-   [`DefId`]主要命名为“定义”或顶级项。
    -   你可以想到[`DefId`]作为一条非常明确和完整的道路的简称，比如`std::collections::HashMap`.但是，这些路径能够命名在正常情况下不可命名的事物（例如IMPLS），并且它们还包括关于板条箱的额外信息（例如其版本号，因为同一板条箱的两个版本可以共存）。
    -   A [`DefId`]实际上由两部分组成，`CrateNum`（识别板条箱）和`DefIndex`（索引到每个板条箱维护的项目列表中）。
-   [`HirId`]将特定项的索引与该项内的偏移量组合在一起。
    -   关键点[`HirId`]是这样吗*相对的*某些项目（通过[`DefId`]）
-   [`BodyId`]，这是引用板条箱中特定主体（函数或常量的定义）的绝对标识符。它目前实际上是一个“新型的”[`NodeId`].
-   [`NodeId`]，它是一个绝对ID，用于标识HIR树中的单个节点。
    -   虽然它们仍在普遍使用，**他们正逐渐被淘汰**.
    -   由于它们在板条箱中是绝对的，因此在树的任何位置添加一个新节点会导致[`NodeId`]更改板条箱中所有后续代码。这对于增量编译是很糟糕的，正如您可能想象的那样。

[`defid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/def_id/struct.DefId.html

[`hirid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.HirId.html

[`bodyid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.BodyId.html

[`nodeid`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.NodeId.html

### HIR地图

大多数情况下，当您与HIR合作时，您将通过**高分辨红外光谱图**，可通过TCX访问[`tcx.hir_map`]（并在[`hir::map`]模块）。这个[高分辨红外光谱图]包含一个[方法的数量]在不同类型的ID之间转换，并查找与HIR节点关联的数据。

[`tcx.hir_map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/context/struct.GlobalCtxt.html#structfield.hir_map

[`hir::map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/index.html

[hir map]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html

[number of methods]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#methods

例如，如果您有[`DefId`]，您想将其转换为[`NodeId`]，你可以使用[`tcx.hir.as_local_node_id(def_id)`][as_local_node_id]. 返回一个`Option<NodeId>`–这将是`None`如果def id引用了当前板条箱之外的某个对象（从那时起它就没有hir节点），则返回`Some(n)`在哪里？`n`是定义的节点ID。

[as_local_node_id]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.as_local_node_id

同样，您可以使用[`tcx.hir.find(n)`][find]查找节点的步骤[`NodeId`]. 返回一个`Option<Node<'tcx>>`在哪里[`Node`]是在映射中定义的枚举；通过对此进行匹配，可以找出节点ID引用的节点类型，还可以获取指向数据本身的指针。通常，您知道什么样的节点`n`例如，如果你知道`n`一定是个hir表达式，你可以[`tcx.hir.expect_expr(n)`][expect_expr]，将提取并返回[`&hir::Expr`][expr]惊慌失措的`n`实际上不是表达式。

[find]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.find

[`node`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/enum.Node.html

[expect_expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.expect_expr

[expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Expr.html

最后，您可以使用hir映射通过如下调用找到节点的父节点[`tcx.hir.get_parent_node(n)`][get_parent_node].

[get_parent_node]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.get_parent_node

### 希尔体

一[`hir::Body`]表示某种可执行代码，例如函数体/闭包或常量定义。实体与**主人**，通常是某种物品（例如`fn()`或`const`，但也可以是一个闭包表达式（例如`|x, y| x + y`）您可以使用hir映射查找与给定def id关联的主体（[`maybe_body_owned_by`]）或者找到尸体的主人（[`body_owner_def_id`]）

[`hir::body`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Body.html

[`maybe_body_owned_by`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.maybe_body_owned_by

[`body_owner_def_id`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.body_owner_def_id
