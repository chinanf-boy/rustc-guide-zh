# 查询：需求驱动的编译

如上所述[编译器的高级概述][hl]Rust 编译器目前正从传统的“基于过程”的设置过渡到“需求驱动”的系统。**编译器查询系统是我们新的需求驱动组织的关键。**这个想法很简单。您有各种查询来计算有关输入的内容，例如，有一个名为`type_of(def_id)`如果给定某个项目的 def id，它将计算该项目的类型并将其返回给您。

[hl]: high-level-overview.zh.html

查询执行为**回忆录**–因此，第一次调用查询时，它将进行计算，但下一次，结果将从哈希表返回。此外，查询执行非常适合**增量计算**；其基本思想是，在执行查询时，**可以**通过从磁盘加载存储的数据返回给您（但这是一个单独的主题，我们在此不再讨论）。

总体设想是，最终，整个编译器控制流将由查询驱动。实际上，有一个顶级查询（“编译”）将在板条箱上运行编译；这反过来将要求从*结束*.例如：

- 这个“编译”查询可能需要获得代码生成单元的列表（即需要由 LLVM 编译的模块）。
- 但是计算 codegen 单元的列表会调用一些子查询，这些子查询返回在 rust 源中定义的所有模块的列表。
- 这个查询反过来会调用一些请求 HIR 的东西。
- 在我们进行实际的解析之前，这一过程会越来越远。

然而，这一愿景尚未完全实现。尽管如此，编译器的大块（例如，生成 mir）工作方式与此完全相同。

### 详细的查询评估模型

这个[详细查询评估模型][query-model]第章更深入地描述了什么是查询以及它们是如何工作的。如果你打算自己写一个查询，这是一个很好的阅读。

### 调用查询

调用查询很简单。TCX（“类型上下文”）为每个定义的查询提供一个方法。例如，调用`type_of`查询，您只需执行以下操作：

```rust,ignore
let ty = tcx.type_of(some_def_id);
```

### 编译器如何执行查询

因此，您可能想知道调用查询方法时会发生什么。答案是，对于每个查询，编译器都会维护一个缓存——如果您的查询已经执行过，那么答案很简单：我们将返回值从缓存中克隆出来并返回它（因此，您应该尝试确保查询的返回类型是可低成本克隆的；插入一个`Rc`如有必要）。

#### 提供者

但是，如果查询是*不*在缓存中，编译器将尝试找到合适的**供应商**. 提供程序是一个已经定义并链接到编译器中的函数，其中包含计算查询结果的代码。

**供应商按板条箱定义。**编译器在内部为每个板条箱维护一个提供者表，至少在概念上是这样的。现在，实际上有两个集合：关于**本地板条箱**（即正在编译的）和提供有关**外箱**（即本地板条箱的依赖性）。注意，决定查询目标的板条箱不是*友善的*但是，_钥匙_. 例如，当您调用`tcx.type_of(def_id)`，这可能是本地查询或外部查询，具体取决于`def_id`指（见`self::keys::Key`关于如何工作的更多信息。

提供程序始终具有相同的签名：

```rust,ignore
fn provider<'cx, 'tcx>(tcx: TyCtxt<'cx, 'tcx, 'tcx>,
                       key: QUERY_KEY)
                       -> QUERY_RESULT
{
    ...
}
```

提供程序接受两个参数：`tcx`和查询键。还请注意，他们接受*全球的*TCX（即他们使用`'tcx`生存期为两次），而不是使用带有一些活动推理上下文的 TCX。它们返回查询结果。

#### 如何设置提供程序

创建 TCX 时，其创建者使用`Providers`结构。这个结构是由这里的宏生成的，但它基本上是一个函数指针的大列表：

```rust,ignore
struct Providers {
    type_of: for<'cx, 'tcx> fn(TyCtxt<'cx, 'tcx, 'tcx>, DefId) -> Ty<'tcx>,
    ...
}
```

目前，我们有一份本地板条箱和一份外部板条箱的结构副本，尽管我们的计划是最终可能每个板条箱有一个。

这些`Provider`结构最终由`librustc_driver`但它通过将工作分布到另一个`rustc_*`板条箱这是通过调用`provide`功能。这些函数看起来像这样：

```rust,ignore
pub fn provide(providers: &mut Providers) {
    *providers = Providers {
        type_of,
        ..*providers
    };
}
```

也就是说，他们`&mut Providers`在适当的地方变异。通常我们使用上面的配方只是因为它看起来不错，但你也可以这样做`providers.type_of = type_of`，相当于。（这里，`type_of`将是一个顶级函数，正如我们之前看到的那样定义。）因此，如果我们想为其他查询添加一个提供者，我们可以调用它`fubar`在上面的板条箱中，我们可以修改`provide()`功能如下：

```rust,ignore
pub fn provide(providers: &mut Providers) {
    *providers = Providers {
        type_of,
        fubar,
        ..*providers
    };
}

fn fubar<'cx, 'tcx>(tcx: TyCtxt<'cx, 'tcx>, key: DefId) -> Fubar<'tcx> { ... }
```

注意，大部分`rustc_*`板条箱仅提供**本地供应商**. 几乎所有**外部提供程序**最后通过[`rustc_metadata`机箱][rustc_metadata]从板条箱元数据加载信息。但在某些情况下，有一些板条箱提供查询*二者都*本地和外部板条箱，在这种情况下，它们定义了`provide`和 A`provide_extern`功能即`rustc_driver`可以调用。

[rustc_metadata]: https://github.com/rust-lang/rust/tree/master/src/librustc_metadata

### 添加新的查询

所以假设您想要添加一种新的查询，您是如何添加的呢？定义查询分两步进行：

1.  首先，必须指定查询名称和参数；然后，
2.  您必须在需要时提供查询提供程序。

要指定查询名称和参数，只需在[`src/librustc/ty/query/mod.rs`][query-mod]，看起来像：

[query-mod]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/query/index.html

```rust,ignore
define_queries! { <'tcx>
    /// Records the type of every item.
    [] fn type_of: TypeOfItem(DefId) -> Ty<'tcx>,

    ...
}
```

宏的每一行定义一个查询。名字是这样分开的：

```rust,ignore
[] fn type_of: TypeOfItem(DefId) -> Ty<'tcx>,
^^    ^^^^^^^  ^^^^^^^^^^ ^^^^^     ^^^^^^^^
|     |        |          |         |
|     |        |          |         result type of query
|     |        |          query key type
|     |        dep-node constructor
|     name of query
query flags
```

让我们逐一检查一下：

- **查询标志：**现在这些基本上是未使用过的，但是目的是我们能够定制查询处理方式的各个方面。
- **查询名称：**查询方法的名称（`tcx.type_of(..)`）也用作结构的名称（`ty::queries::type_of`）将生成以表示此查询的。
- **DEP 节点构造函数：**指示将此查询连接到增量编译的构造函数函数。通常，这是`DepNode`变量，可通过修改`define_dep_nodes!`宏调用[`librustc/dep_graph/dep_node.rs`][dep-node].
  - 但是，有时我们使用自定义函数，在这种情况下，名称将在 snake 中，函数将在文件的底部定义。这通常在查询键不是 def id 或只是 dep 节点所期望的类型时使用。
- **查询关键字类型：**此查询的参数类型。此类型必须实现`ty::query::keys::Key`特征，它定义（例如）如何将其映射到板条箱，等等。
- **查询结果类型：**此查询生成的类型。此类型不应（a）使用`RefCell`或其他内部变异性和（b）是廉价的克隆。实习或使用`Rc`或`Arc`建议用于重要的数据类型。
  - 这些规则的一个例外是`ty::steal::Steal`类型，用于廉价地就地修改 mir。参见的定义`Steal`了解更多详细信息。新用途`Steal`应该**不**添加时不发出警报`@rust-lang/compiler`.

[dep-node]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/dep_graph/struct.DepNode.html

因此，要添加查询：

- 添加条目`define_queries!`使用上述格式。
- 可能会向 DEP NODE 宏添加相应的条目。
- 通过修改适当的`provide`方法；或在需要时添加新方法并确保`rustc_driver`正在调用它。

#### 查询结构和说明

对于每种类型，`define_queries`宏将生成以查询命名的“查询结构”。这个结构是一种描述查询的占位符。每个这样的结构实现`self::config::QueryConfig`特征，它为该特定查询的键/值关联了类型。基本上，生成的代码如下所示：

```rust,ignore
// Dummy struct representing a particular kind of query:
pub struct type_of<'tcx> { phantom: PhantomData<&'tcx ()> }

impl<'tcx> QueryConfig for type_of<'tcx> {
  type Key = DefId;
  type Value = Ty<'tcx>;
}
```

您可能希望实现一个附加特性，调用`self::config::QueryDescription`. 这个特性在循环错误期间使用，为查询提供一个“人类可读”的名称，这样我们就可以总结循环发生时发生了什么。如果查询键是`DefId`但是如果你*不要*实现它，你会得到一个非常普通的错误（“处理`foo`……“。您可以将新的 IMPLS 放入`config`模块。它们看起来像这样：

```rust,ignore
impl<'tcx> QueryDescription for queries::type_of<'tcx> {
    fn describe(tcx: TyCtxt, key: DefId) -> String {
        format!("computing the type of `{}`", tcx.item_path_str(key))
    }
}
```

[query-model]: queries/query-evaluation-model-in-detail.html
