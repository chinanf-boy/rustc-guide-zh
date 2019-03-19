# 查询：需求驱动的编译

如[编译器的天台风景][hl]上所述的那样，Rust 编译器目前正从传统的“基于过程”设置过渡到“需求驱动”系统。**编译器查询系统是我们新的需求驱动管理的关键。**想法很简单。您有各种查询来计算关于输入的内容，例如，有一个`type_of(def_id)`的查询，给出 def_id 参数项，它将计算该项的类型，并将其返回给您。

[hl]: high-level-overview.zh.html

查询的执行是**有记性的** —— 因此，在第一次调用查询时，它会进行计算，但下一次，结果就从一个哈希表返回。此外，查询执行非常适合**增量计算**；其基本思想是，在执行查询时，返回的结果**可以**是，从磁盘加载的存储数据（但这是一个独立的主题，我们在此不再讨论）。

总体设想是，最终，整个编译器控制流将由查询驱动。由一个顶层查询进行有效控制（“编译”），就是在一个箱子上运行编译；相应的，会需要关于这箱子的信息，在*结束*时会用到。例如：

- 这个“编译”查询可能需要获得 codegen-单元 的列表（即，需要由 LLVM 编译的模块）。
- 但是计算 codegen 单元的列表会调用一些子查询，这些子查询返回在 rust 源代码中定义的所有模块的列表。
- 这个查询依次会调用一些请求 HIR 的东西。
- 在我们进行实际的解析之前，这一过程会来来去去。

然而，这一愿景尚未完全实现。不过，编译器的大部分（例如，生成 MIR）工作方式与此完全相同。

### 详细的查询执行模型

这个[详细的查询执行模型][query-model]章节，更深入地描述了什么是查询以及它们是如何工作的。如果你打算自己写一个查询，这是一个很好的读物。

### 调用查询

调用查询很简单。tcx（“类型上下文{type context}”）为每个定义的查询提供一个方法。例如，要调用`type_of`查询，您只需执行以下操作：

```rust,ignore
let ty = tcx.type_of(some_def_id);
```

### 编译器如何执行查询

您可能想知道调用查询方法时，会发生什么。答案是，对于每个查询，编译器都会维护一个缓存 —— 如果您的查询已经执行过，那么答案很简单：我们的返回值是从缓存中克隆出来的（因此，您应该尝试，确保查询的返回类型是可低成本克隆的；如有必要的话，插入一个`Rc`）。

#### Provider

> 提供，供给者

但是，如果查询*不*在缓存中，编译器将尝试找到合适的**Provider**。 Providers 是一个已经定义，并链接到编译器中的函数，其中包含计算查询结果的代码。

**Provider 按每个箱子定义。**编译器在内部为每个箱子维护一个 provider 表，至少在概念上是这样的。现在，实际上有两个集合：关于**本地箱子**（即，正在编译的）和关于**外部箱**查询的 Provider（即，本地箱子的依赖）。注意，查询的箱子目标，不取决与查询*类型*，而在于 _key_。 例如，当您调用`tcx.type_of(def_id)`，这可能是本地查询或外部查询，具体取决于，箱子`def_id`的指向（见`self::keys::Key`trait，关于其如何工作的更多信息）。

Providers 始终具有相同的签名：

```rust,ignore
fn provider<'cx, 'tcx>(tcx: TyCtxt<'cx, 'tcx, 'tcx>,
                       key: QUERY_KEY)
                       -> QUERY_RESULT
{
    ...
}
```

Providers 接受两个参数：`tcx`和查询 key。还请注意，他们接受*全局的*tcx（即他们使用`'tcx`的生命周期，两次），而不是使用一个，处于一些活动上下文的 tcx。它们返回查询结果。

#### 如何设置 Providers

当创建 tcx ，其创建函数使用`Providers`结构，给予了 providers。这个结构是由这里的宏生成的，但它基本上是一个函数指针的大列表：

```rust,ignore
struct Providers {
    type_of: for<'cx, 'tcx> fn(TyCtxt<'cx, 'tcx, 'tcx>, DefId) -> Ty<'tcx>,
    ...
}
```

目前，我们有一份本地箱子和一份外部箱子的结构副本(copy)，尽管我们的计划，最终可能每个箱子都有一个。

这些`Provider`结构最终由`librustc_driver`创建和支持，但它会把工作分布到其他`rustc_*`箱子。通过调用多个`provide`函数来完成，这些函数看起来像这样：

```rust,ignore
pub fn provide(providers: &mut Providers) {
    *providers = Providers {
        type_of,
        ..*providers
    };
}
```

也就是说，他们拿到了`&mut Providers`并在适当的地方修改它。通常我们使用上面的配方，只是因为它看起来不错，但你也可以用`providers.type_of = type_of`，是等价的。（这里，`type_of`将是一个顶层函数，正如我们之前看到的那样定义。）因此，如果我们想为其他查询添加一个 provider，会去调用`fubar`，它进入上面的箱子。就这样，我们可以修改`provide()`函数：

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

注意，大部分`rustc_*`箱子仅提供**本地 Provider**。 几乎所有的**外部 Providers**都会经过，从箱子元数据加载信息的[`rustc_metadata`箱子][rustc_metadata]，才会使用。但也会存在一些箱子为本地*和*外部箱子提供查询，在这种情况下，它们定义了`provide`和一个`provide_extern`函数，让`rustc_driver`可以调用。

[rustc_metadata]: https://github.com/rust-lang/rust/tree/master/src/librustc_metadata

### 添加新的查询

接下来，假设您想要添加一种新的查询，要如何添加呢？定义查询分两步进行：

1.  首先，必须指定查询名称和参数；然后，
2.  您必须在需要时，提供查询 Providers。

要指定查询名称和参数，只需在[`src/librustc/ty/query/mod.rs`][query-mod]添加一个条目，给大型宏命令调用，看起来像：

[query-mod]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/query/index.html

```rust,ignore
define_queries! { <'tcx>
    /// 记录每个 item 的类型.
    [] fn type_of: TypeOfItem(DefId) -> Ty<'tcx>,

    ...
}
```

宏的每一行，定义一个查询。名字是像这样分开的：

```rust,ignore
[] fn type_of: TypeOfItem(DefId) -> Ty<'tcx>,
^^    ^^^^^^^  ^^^^^^^^^^ ^^^^^     ^^^^^^^^
|     |        |          |         |
|     |        |          |         查询的结果类型
|     |        |          查询 key 类型
|     |        dep-node 构造函数
|     查询名称
查询标志
```

让我们逐一过一遍：

- **查询标志(Query flags)：**现在这里，我们还没用，但是目的是我们能够定制查询处理的各个方面。
- **查询名称(Name of query)：**查询方法的名称（`tcx.type_of(..)`）。也用作一个生成的结构名称（`ty::queries::type_of`），来表示此查询。
- **Dep-node 构造函数(Dep-node constructor)：**指示将此查询连接到增量编译的构造函数。通常，这是`DepNode`变种，可通过在[`librustc/dep_graph/dep_node.rs`][dep-node]中，修改`define_dep_nodes!`宏调用，达到添加的目的。
  - 但是有时，我们会使用一个自定义函数，在这种情况下，名称要是 snake 形式，和该函数定义在文件的底部。这通常在查询 key 不是 _def-id_ 或不是 _dep-node_ 所期望的类型时，使用。
- **查询键类型(Query key type)：**此查询的参数类型。此类型必须实现`ty::query::keys::Key`trait，它定义了（例如）如何将其自身映射到箱子的操作，等等。
- **查询结果类型(Result type of query)：**此查询生产的类型。此类型不应：（1）使用`RefCell`或其他内部可变性；（2）是可廉价克隆。Interning 或使用`Rc`或`Arc`建议，用于重要的数据类型。
  - 一个例外是`ty::steal::Steal`类型，用于就地廉价修改 MIR。参见`Steal`的定义，了解更多详细信息。添加`Steal`的新用法**不**应该，不发出`@rust-lang/compiler`警报。

[dep-node]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/dep_graph/struct.DepNode.html

总结，要添加一个查询：

- 使用上述格式，给`define_queries!`添加一个条目。
- 可能会向 dep-node 宏添加相应的条目。
- 通过修改适当的`provide`方法，接上 provider；或在需要时添加一个新的方法，并确保`rustc_driver`在调用它。

#### 查询结构和说明

对于每种类型，`define_queries`宏会生成，以查询命名的“查询结构”。这个结构是一种描述查询的占位方式。每一个这样的结构，都实现了`self::config::QueryConfig`trait，它为特定查询的 键/值 关联上类型。基本上，生成的代码如下所示：

```rust,ignore
// 代表一个某种特定查询的，伪结构:
pub struct type_of<'tcx> { phantom: PhantomData<&'tcx ()> }

impl<'tcx> QueryConfig for type_of<'tcx> {
  type Key = DefId;
  type Value = Ty<'tcx>;
}
```

您若是想，实现一个附加 trait，就调用`self::config::QueryDescription`。这个 trait 在循环错误期间使用，为查询提供一个“人类可读”的名称，这样我们就可以概述下，在循环发生了什么。如果查询键(key)是`DefId`，那么 trait 的实现是可选的，但是如果你*不*实现它，你会得到一个非常普通的错误（"processing`foo`……")。

您可以将新的实现区块，放入`config`模块。它们看起来像这样：

```rust,ignore
impl<'tcx> QueryDescription for queries::type_of<'tcx> {
    fn describe(tcx: TyCtxt, key: DefId) -> String {
        format!("computing the type of `{}`", tcx.item_path_str(key))
    }
}
```

[query-model]: queries/query-evaluation-model-in-detail.zh.html
