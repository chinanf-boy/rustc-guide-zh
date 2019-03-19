# HIR

HIR —— “高级中间描述（High-Level Intermediate Representation）” —— 是 rustc 大多数使用的主要 IR。它是在解析、宏扩展和名称解析之后，生成对编译器友好的抽象语法树（ast）描述（请参见[降层(Lowering)](./lowering.zh.html)，关于如何创建 HIR）。hir 的许多部分与 rust surface 语法非常相似，只是 rust 的一些表达形式已经被删除了。例如，`for`循环会转换为一个`loop`，且不出现在 HIR 中。这使得 hir 比正常的 ast 更易于分析。

本章介绍了 HIR 的主要概念。

您可以通过传递`-Zunpretty=hir-tree`标志，给 rustc ：

```bash
> cargo rustc -- -Zunpretty=hir-tree
```

### 外带的存储和`Crate`类型

HIR 中的顶级数据结构是[`Crate`]，它存储当前正在编译箱子的内容（我们只为当前箱子构建 HIR）。而在 AST 中，箱子数据结构基本上只包含根模块， HIR 的`Crate`结构包含许多 map（数据容器） 和其他东西，用于组织箱子的内容，以便于访问。

[`crate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Crate.html

例如，HIR 中单个项的内容（例如模块、函数、trait、impl 等）在父级中，不能立即访问。如若存在，一个模块项`foo`，其包含一个函数`bar()`：

```rust
mod foo {
    fn bar() { }
}
```

然后在 HIR 中，`foo`模块的描述 ([`Mod`]结构）将只具有表示 `bar()`的`I` **`ItemId`** (编号)。套获取`bar()`函数的详细信息，我们会在`items`map，查找`I`。

[`mod`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Mod.html

这种描述的一个很好的结果是，通过迭代这些映射中的键值对，可以迭代箱子中的所有项（而不需要拖拽整个 HIR）。对于 trait 项和 impl 项，以及“body”们（下面会解释到）都有类似的 map 。

以这种方式设置这种描述的另一个原因，是为了更好地与增量编译集成。这样，如果您可以获取对[`&hir::Item`]的访问（例如，mod `foo`），你不能立即访问到`bar()`函数的内容。相反，您只能访问`bar()`的**id**，且必须调用一些函数，用给定的 id 来查找`bar()`；这使编译器有机会观察，您访问`bar()`数据过程，和记录其依赖项。

[`&hir::item`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Item.html

<a name="hir-id"></a>

### HIR 中的标识符

大多数在 HIR 中，需要处理事物的代码，更倾向不将引用带到 HIR 中，而是带到 _标识符编号_（或，叫做“id”）。现在，您将发现四种正在使用的标识符：

- [`DefId`]，主要为“definitions(描述)”或顶级项命名。
  - 你可以把一个[`DefId`]想象成，一条非常明确和完整道路的简称，比如`std::collections::HashMap`。但是，这些路径也能够命名在正常情况下，不可命名的事物（例如：impl），并且它们还包括关于箱子的额外信息（例如其版本号，因为同一箱子的两个版本，可以共存）。
  - 体格[`DefId`]实际上由两部分组成，`CrateNum`（识别箱子）和`DefIndex`（索引到，维护每个箱子的项目列表中）。
- [`HirId`]，将特定项的索引，与该项内的偏移量组合在一起。
  - [`HirId`]的关键点是，与*某些项*（由一个[`DefId`]命名）是相对的
- [`BodyId`]，这是绝对标识符，引用箱子中，特定主体（函数或常量的定义）。目前，它实际上是一个“新型的”[`NodeId`].
- [`NodeId`]，它是一个绝对 ID，用于标识 HIR 树中的单个节点。
  - 虽然它们仍在普遍使用，**他们正逐渐被淘汰**.
  - 由于它们在箱子中是绝对的，因此在树的任何位置添加一个新节点，会导致箱子中，所有后续代码的[`NodeId`]更改。这对增量编译是很糟糕的，正如您可能想象的那样(坍塌)。

[`defid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/def_id/struct.DefId.html
[`hirid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.HirId.html
[`bodyid`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.BodyId.html
[`nodeid`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.NodeId.html

### HIR Map

大多数情况下，当您与 HIR 合作时，**HIR Map**是你的得力助手，可通过 [`tcx.hir_map`]（[`hir::map`]模块中定义），在 tcx 中访问。这个[HIR Map]包含一[定数量的方法]，能在不同类型的 ID 之间转换，并查找与 HIR 节点关联的数据。

[`tcx.hir_map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/context/struct.GlobalCtxt.html#structfield.hir_map
[`hir::map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/index.html
[hir map]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html
[number of methods]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#methods

例如，如果您有一个[`DefId`]，您想将其转换为[`NodeId`]，你可以使用[`tcx.hir.as_local_node_id(def_id)`][as_local_node_id]，这会返回一个`Option<NodeId>` —— 如果 def-id 引用了当前箱子之外的某个对象（从那时起，它就没有 HIR 节点），会是`None`，否则返回`Some(n)`，这里的`n`是定义的 node-id。

[as_local_node_id]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.as_local_node_id

同样，您可以使用[`tcx.hir.find(n)`][find]查找一个[`NodeId`]的节点，返回一个`Option<Node<'tcx>>`，这里的[`Node`]是在 map 中定义的枚举；通过对此进行匹配，可以找出 node-id 引用的节点类型，还可以获取指向数据本身的指针。通常，您知道节点`n`是什么样的 —— 例如，如果你知道`n`一定是个 HIR 表达式，你可以用[`tcx.hir.expect_expr(n)`][expect_expr]提取并返回[`&hir::Expr`][expr]，但若`n`实际上不是表达式，会 panic。

[find]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.find
[`node`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/enum.Node.html
[expect_expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.expect_expr
[expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Expr.html

最后，您可以使用 HIR map，找到节点的父节点，通过如下调用，像[`tcx.hir.get_parent_node(n)`][get_parent_node].

[get_parent_node]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.get_parent_node

### HIR Bodies(主体)

> owner 虽然是所有者的意思，但 Rust 所有权术语更为准确

一个[`hir::Body`]表示某种可执行代码，例如一个函数体/闭包，或常量定义主体，而 Bodies 会分配一个**所有权**，通常是某类项（例如`fn()`或`const`，但也可以是一个闭包表达式（例如`|x, y| x + y`），您可以用 HIR map 查找给定 def-id([`maybe_body_owned_by`])，所关联的主体，或者找到（[`body_owner_def_id`]）主体的所有权。

[`hir::body`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Body.html
[`maybe_body_owned_by`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.maybe_body_owned_by
[`body_owner_def_id`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/map/struct.Map.html#method.body_owner_def_id
