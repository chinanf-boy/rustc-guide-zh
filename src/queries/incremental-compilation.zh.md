# 增量编译

增量编译方案，本质上是对整个查询系统的一个简单却又惊喜的扩展。首先，我们描述一个稍微简化的事实——“基本算法”，然后描述一些可能的改进。

## 基本算法

基本算法称为**红绿色**算法[^salsa]。主要的想法是，在每次运行编译器之后，我们将保存我们所做的所有查询结果，以及**查询的 DAG**。 这个**查询 DAG**是一个[DAG]，是查询及其执行其他查询的索引图。因此，例如，如果计算 Q1 需要计算 Q2，那么从查询 Q1 到另一个查询 Q2 会有一条边（注意，由于查询不能依赖于它们本身，所以这会导致 DAG 而不是通用图{general graph}）。

[dag]: https://en.wikipedia.org/wiki/Directed_acyclic_graph

在编译器的下一次运行中，我们有时可以重用这些查询结果，以避免重新执行查询。具体做法是，我们通过分配每个查询，一个**颜色**：

- 如果查询是**红色**，这意味着它在编译期间的结果，与以前的编译相比**改变了**。
- 如果查询是**绿色**，这意味着它的结果，与以前的编译相比是**相同的**。

这里有两个关键的见解：

- 第一，如果查询 Q 的所有输入都是绿色的，那么查询 Q 的结果**必然(需)**与上次的值相同，因此不需要重新执行（否则编译器就不具有确定性）。
- 第二，即使查询的某些输入发生更改，也可能是**仍**产生与先前编译相同的结果。特别是，若查询只使用输入的一部分。
  - 因此，在执行查询之后，我们总是会检查它，是否产生与前一次相同的结果。**如果是，**我们仍然可以将查询标记为绿色，从而避免重新执行依赖(Q)的查询。

### try-mark-green 算法

增量编译的核心是一个叫"try-mark-green"的算法。它的任务是确定，给定查询 Q 的颜色（必须还没执行）。在 Q 有红色输入的情况下，Q 颜色的确定，可能需要 Q 去重新执行，以便我们可以比较它的输出；但是如果 Q 的所有输入都是绿色的，那么我们可以得出结论，Q 必然是绿色的，而不需要重新执行或检查它的值。在编译器中，这允许我们避免在不需要时，从磁盘反序列化结果，事实不止如此，我们有时还可以跳过*序列化*结果（请参见下面的“改进(refinements)”小节）。

Try-mark-green，按如下方式工作：

- 首先检查查询 Q 在上次编译期间，是否执行了。
  - 如果没有，我们可以正常地重新执行查询，并将其指定为红色。
- 如果是，则加载 Q 的“依赖查询”。
- 如果有保存的结果，则加载来自查询 DAG(图/列表) 的`reads(Q)`向量。“reads”是 Q 在执行期间，启用的一组查询。
  - 对于`reads(Q)`中的每个查询 R，我们使用 try-mark-green 递归地确认 R 的颜色。
    - 注意：重要的是，我们访问`reads(Q)`与原始编译中的顺序相同。见[下面关于，查询 DAG 的小节](#dag).
    - 如果`reads(Q)`中，**任何**节点为**红色**，则 Q 是脏的。
      - 我们重新执行 Q 并将其结果的散列，与前一次编译的结果的散列进行比较。
      - 如果散列值没有更改，我们可以将 Q 标记为**绿色**，然后返回。
    - 否则，`reads(Q)`中，**全部的**节点中的必须是**绿色**。 在这种情况下，我们可以将 Q 着色为**绿色**，然后返回。

<a name="dag"></a>

### 查询 DAG

查询 DAG 的代码存储在[`src/librustc/dep_graph`][dep_graph]。 DAG 构造是利用查询执行来完成的。

一个关键点是查询 DAG 还跟踪顺序；也就是说，对于每个查询 Q，我们不仅跟踪 Q 读取的查询，还跟踪这些查询读取的**顺序**。这允许 try-mark-green 以相同的顺序返回这些查询。这一点很重要，因为一旦子查询恢复为红色，我们就不能再确定 Q ，是否会像以前一样，(在 DAG)沿着相同的路径继续。也可以想象，这样一个查询：

```rust,ignore
fn main_query(tcx) {
    if tcx.subquery1() {
        tcx.subquery2()
    } else {
        tcx.subquery3()
    }
}
```

现在想象一下，在第一次编辑时，`main_query`从执行`subquery1`开始，这将返回 true。在这种情况下，`main_query`查询，下一个会执行`subquery2`，而`subquery3`不会被执行。

但是现在想象一下，**下一个**编译时，输入已改为`subquery1`返回**false**，在这种情况下，`subquery2`永远不会执行。如果 try-mark-green 不按照顺序，就去执行`reads(main_query)`，那么，有可能出现在`subquery1`之前，就执行`subquery2`的情况。导致编译器的 ICEs 和其他问题。

[dep_graph]: https://github.com/rust-lang/rust/tree/master/src/librustc/dep_graph

## 基本算法的改进

在对基本算法的描述中，我们说在编译结束时，我们将保存执行的所有查询结果。在实践中，这可能是件非常浪费的事情 —— 许多结果的重新计算是非常便宜的，而序列化和反序列化，并没有大胜之。实际上，我们要做的就是保存执行过的所有子查询的**哈希**。同样的，在某些情况下，我们**也**保存结果。

这就是增量算法的提升，为什么要分离节点**颜色**的计算的原因，是因为通常，要得到该值不用通过计算一个节点的**结果**来得到。计算颜色结果是，通过下方一个简单的算法完成的：

- 检查保存的 Q 结果是否可用。如果能，则计算 Q 的颜色。如果 Q 为绿色，则反序列化，并返回保存的结果。
- 否则，执行 Q 查询。
  - 然后，我们可以比较结果的哈希值，如果结果没有改变，那么颜色 Q 将变为绿色。

## 资源

初始的设计文件，在[https://github.com/nikomastsakis/rustc-on-demand-incremental-design-doc/blob/master/0000-rustc-on-demand-and-incremental.md](https://github.com/nikomatsakis/rustc-on-demand-incremental-design-doc/blob/master/0000-rustc-on-demand-and-incremental.md)，它扩展了记忆化细节，为这个系统提供了更高层次的概述和动力。

# 脚注

[^salsa]：我很早就想将其重命名为 salsa 算法，但它从未流行过。-@nikomatsakis
