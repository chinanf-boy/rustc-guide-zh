# 米尔通行证

如果您想获取函数（或常量等）的mir，可以使用`optimized_mir(def_id)`查询。这将返回最终优化的mir。对于国外的def id，我们只需从另一个板条箱的元数据中读取mir。但是对于本地def id，查询将构造mir，然后通过应用一系列过程对其进行迭代优化。本节介绍这些过程如何工作以及如何扩展它们。

生产`optimized_mir(D)`对于给定的def id`D`，mir通过几个优化套件，每个优化套件由一个查询表示。每个套件包含多个优化和转换。这些套件代表了有用的中间点，我们希望访问mir进行类型检查或其他目的：

-   `mir_build(D)`–不是查询，但这构造了初始mir
-   `mir_const(D)`–应用一些简单的转换，使MIR准备好进行持续评估；
-   `mir_validated(D)`–应用更多的转换，使mir准备好进行借用检查；
-   `optimized_mir(D)`–执行所有优化后的最终状态。

### 实施和注册通行证

一`MirPass`是一些处理mir的代码，通常（但不总是）以某种方式转换它。例如，它可能执行优化。这个`MirPass`性状本身存在于[这个`rustc_mir::transform`模块][mirtransform]基本上由一种方法组成，`run_pass`，只需`&mut Mir`（连同TCX和它来自哪里的一些信息）。因此，MIR被修改到位（这有助于保持效率）。

基本mir传递的一个好例子是[`NoLandingPads`]，移动mir并删除由于展开而导致的所有边缘–当配置为`panic=abort`从不放松。从源代码中可以看到，mir过程是通过首先定义一个虚拟类型、一个不带字段的结构来定义的，类似于：

```rust
struct MyPass;
```

然后为其实现`MirPass`特质。然后，您可以将此过程插入到查询中找到的相应过程列表中，如`optimized_mir`，请`mir_validated`等等（如果这是一个优化，它应该进入`optimized_mir`列表。）

如果你正在写通行证，你很有可能会使用[MIR访客].mir访问者是一种方便的方式，可以浏览mir的所有部分，或者搜索某些内容，或者进行小的编辑。

### 偷窃

中间查询`mir_const()`和`mir_validated()`提高产量`&'tcx Steal<Mir<'tcx>>`，分配使用`tcx.alloc_steal_mir()`. 这表明结果可能是**偷**通过下一套优化——这是一个避免克隆mir的优化。尝试使用被盗结果将导致编译器出现恐慌。因此，除了作为mir处理管道的一部分之外，不要直接从这些中间查询中读取数据，这一点很重要。

由于这种窃取机制，还必须注意确保在处理管道的特定阶段的MIR被窃取之前，任何想从中读取的人都已经这样做了。具体来说，这意味着如果您有一些查询`foo(D)`想要获得`mir_const(D)`或`mir_validated(D)`你需要让继任者通过“武力”`foo(D)`使用`ty::queries::foo::force(...)`. 这将强制执行查询，即使您不直接要求其结果。

例如，考虑mir const限定。它想读取`mir_const()`一套。然而，结果将是**偷**由`mir_validated()`一套。如果什么都没做，那么`mir_const_qualif(D)`如果它以前就成功了`mir_validated(D)`，否则失败。因此，`mir_validated(D)`将**力** `mir_const_qualif`在它真正窃取之前，确保读取已经发生（记住[查询已记忆](../query.html)，因此执行两次查询只需第二次从缓存加载）：

```text
mir_const(D) --read-by--> mir_const_qualif(D)
     |                       ^
  stolen-by                  |
     |                    (forces)
     v                       |
mir_validated(D) ------------+
```

这种机制有点不可靠。在[锈色-锈色41710/].

[rust-lang/rust#41710]: https://github.com/rust-lang/rust/issues/41710

[mirtransform]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/

[`nolandingpads`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/no_landing_pads/struct.NoLandingPads.html

[mir visitor]: ./visitor.html
