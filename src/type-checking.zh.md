# 类型

的[`rustc_typeck`][typeck]Crate含有“类型”和“类型checking”的来源，包括其他相关功能的其他比特。（it raws heavy on the way[类型]和[独奏曲]页：1

[typeck]: https://github.com/rust-lang/rust/tree/master/src/librustc_typeck

[type inference]: type-inference.html

[trait solving]: traits/resolution.html

## 类型

“收藏品”是在蓝石中转换类型的过程。`hir::Ty`这是一个很好的内容，这是一个很好的内容。**内部介绍**用编译法使用`Ty<'tcx>`-我们也做类似的转换器，以在何处发生。

To try and get a sex for the difference，consider this function：

```rust,ignore
struct Foo { }
fn foo(x: Foo, y: self::Foo) { ... }
//        ^^^     ^^^^^^^^^
```

两个寄生虫`x`和`y`每个都有相同的类型：但它们将具有不同的`hir::Ty`节点。这些节点将具有不同的跨度，当然它们对路径的编码也会有所不同。但一旦它们被“收集”到`Ty<'tcx>`节点，它们将由完全相同的内部类型表示。

集合定义为[查询]用于计算有关正在编译的箱子中的各种功能、特性和其他项的信息。注意，每个查询都与*程序间的*例如，对于函数定义，集合将计算出函数的类型和签名，但它不会访问*身体*以任何方式检查函数，也不要检查局部变量上的类型注释（这是类型的工作*检查*）

有关详细信息，请参阅[`collect`][collect]模块。

[queries]: query.html

[collect]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/collect/

**托多**：实际上讨论类型检查…
