# 这个`ty`模块：表示类型

这个`ty`模块定义Rust编译器如何在内部表示类型。它还定义了*打字上下文*（`tcx`或`TyCtxt`，这是编译器中的中心数据结构。

## TCX及其使用寿命

这个`tcx`（“键入上下文”）是编译器中的中心数据结构。它是用于执行所有查询方式的上下文。结构体`TyCtxt`定义对此共享上下文的引用：

```rust,ignore
tcx: TyCtxt<'a, 'gcx, 'tcx>
//          --  ----  ----
//          |   |     |
//          |   |     innermost arena lifetime (if any)
//          |   "global arena" lifetime
//          lifetime of this reference
```

如你所见，`TyCtxt`类型需要三个生存期参数。这些生命周期可能是理解TCX最复杂的事情。在生锈编译期间，我们将大部分内存分配到**竞技场**，基本上是一次释放所有内存的池。当你看到一个一生都像`'tcx`或`'gcx`，您知道它指的是竞技场分配的数据（或与竞技场一样长的数据，无论如何）。

我们使用两种不同层次的砂。外层是“全球舞台”。这个竞技场将持续整个编译过程：所以只有在编译基本结束后（实际上，当我们转向执行LLVM时），您在其中分配的任何内容才会被释放。

为了减少峰值内存的使用，当我们进行类型推断时，我们还使用了内部级别的竞技场。一旦类型推断结束，这些竞技场就会被丢弃。这样做是因为类型推断会生成许多在类型推断完成后并不特别有趣的“丢弃”类型，因此保留这些分配是浪费的。

通常，我们希望编写明确断言在推理期间不会发生这种情况的代码。在这种情况下，没有“本地”竞技场，您可以访问的所有类型都在全局竞技场中分配。为了表达这一点，我们的想法是在`'gcx`和`'tcx`参数`TyCtxt`. 只是让人有点困惑，我们往往用这个名字`'tcx`在这种情况下。下面是一个例子：

```rust,ignore
fn not_in_inference<'a, 'tcx>(tcx: TyCtxt<'a, 'tcx, 'tcx>, def_id: DefId) {
    //                                        ----  ----
    //                                        Using the same lifetime here asserts
    //                                        that the innermost arena accessible through
    //                                        this reference *is* the global arena.
}
```

相反，如果我们想要在类型推断期间使用的代码，那么您需要声明一个`'gcx`和`'tcx`寿命参数：

```rust,ignore
fn maybe_in_inference<'a, 'gcx, 'tcx>(tcx: TyCtxt<'a, 'gcx, 'tcx>, def_id: DefId) {
    //                                                ----  ----
    //                                        Using different lifetimes here means that
    //                                        the innermost arena *may* be distinct
    //                                        from the global arena (but doesn't have to be).
}
```

### 分配和使用类型

生锈类型用`Ty<'tcx>`定义在`ty`模块（不要与`Ty`结构从[HIR]）实际上，这是用于引用的简单类型别名`'tcx`寿命：

```rust,ignore
pub type Ty<'tcx> = &'tcx TyS<'tcx>;
```

[the hir]: ./hir.html

你基本上可以忽略`TyS`结构——基本上你永远不会明确地访问它。我们总是用`Ty<'tcx>`别名——我认为唯一的例外是定义类型的固有方法。实例`TyS`仅在一个Rustc竞技场中分配（从未在堆栈上分配）。

对类型的一个常见操作是**比赛**看看它们是什么类型的。这是通过做`match ty.sty`，有点像这样：

```rust,ignore
fn test_type<'tcx>(ty: Ty<'tcx>) {
    match ty.sty {
        ty::TyArray(elem_ty, len) => { ... }
        ...
    }
}
```

这个`sty`字段（我不清楚这个名称的来源；可能是结构类型？）属于类型`TyKind<'tcx>`这是一个枚举，定义编译器中所有不同类型的类型。

> 注意检查`sty`类型推断过程中类型上的字段可能有风险，因为可能需要考虑推理变量和其他事项，或者有时类型还不知道，稍后会知道）。

要分配新类型，可以使用`mk_`在上定义的方法`tcx`. 这些名称主要对应于各种类型的变体。例如：

```rust,ignore
let array_ty = tcx.mk_array(elem_ty, len * 2);
```

这些方法都返回`Ty<'tcx>`–请注意，你回来的生命是最深处竞技场的生命`tcx`可以访问。事实上，类型总是被规范化和内部化（因此我们从不分配两次完全相同的类型），并且总是在最外层的区域中分配它们（因此，如果它们不包含任何推理变量或其他“临时”类型，它们将在全局区域中分配）。然而，一生`'tcx`总是一个安全的近似值，所以这就是你能得到的。

> 铌。由于类型是内部的，因此可以使用`==`但是，除非您碰巧在散列和查找重复项，否则这几乎不是您想要做的。这是因为通常在Rust中有多种方法来表示同一类型，特别是在涉及到推理时。如果要测试类型相等性，您可能需要开始研究推理代码来正确地进行测试。

您还可以在`tcx`通过访问`tcx.types.bool`，`tcx.types.char`等`CommonTypes`更多）。

### 超越类型：其他类型的竞技场分配数据结构

除了类型之外，还有许多其他分配给竞技场的数据结构可以分配，这些数据结构可以在本模块中找到。以下是几个例子：

-   [`Substs`][subst]，分配`mk_substs`–这将实习一部分类型，通常用于指定要替代泛型的值（例如`HashMap<i32, u32>`将表示为一个切片`&'tcx [tcx.types.i32, tcx.types.u32]`）
-   `TraitRef`，通常通过值–A传递**性状参考**包括对特征及其各种类型参数的引用（包括`Self`）`i32: Display`（此处，def id将引用`Display`属性，子字符串将包含`i32`）
-   `Predicate`定义了特征系统必须证明的东西（参见`traits`模块）。

[subst]: ./kinds.html#subst

### 导入约定

虽然没有硬性规定，但是`ty`模块的使用方式如下：

```rust,ignore
use ty::{self, Ty, TyCtxt};
```

特别是，由于它们很常见，`Ty`和`TyCtxt`类型直接导入。其他类型通常用显式引用`ty::`前缀（例如）`ty::TraitRef<'tcx>`）但有些模块选择显式导入一组较大或较小的名称。
