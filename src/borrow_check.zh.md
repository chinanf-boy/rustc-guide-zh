# MIR借用支票

借用支票是Rust的“秘密酱” - 它的任务是执行许多属性：

-   所有变量在使用之前都已初始化。
-   你不能两次移动相同的值。
-   在借用时你不能移动一个值。
-   您可以在可变借用的地方访问某个地方（通过参考除外）。
-   在共享借用时你不能改变一个地方。
-   等等

在撰写本文时，代码处于转换状态。“主要”借用检查器仍然通过处理工作[HIR](hir.html)，但正在逐步取消有利于基于MIR的借用检查员。因此，本文档重点介绍新的基于MIR的借用检查程序。

对MIR进行借阅检查有几个好处：

-   MIR是*远*比HIR复杂;激进的贬义有助于防止借入检查者的错误。（如果你很好奇，你可以看到[基于MIR的借用检查程序在此处修复的错误列表][47366]。）
-   更重要的是，使用MIR可以实现[“非词汇生命”][nll]，这是从控制流图得到的区域。

[47366]: https://github.com/rust-lang/rust/issues/47366

[nll]: http://rust-lang.github.io/rfcs/2094-nll.html

### 借款检查员的主要阶段

借阅检查器源位于[该`rustc_mir::borrow_check`模][b_c]。主要切入点是[`mir_borrowck`]查询。

[b_c]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/index.html

[`mir_borrowck`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/fn.mir_borrowck.html

-   我们先创建一个**本地副本**MIR。在接下来的步骤中，我们将修改此副本以修改类型和事物，以包括对我们正在计算的新区域的引用。
-   然后我们调用[`replace_regions_in_mir`]修改我们当地的MIR。除此之外，这个功能将取代所有的[地区](./appendix/glossary.html)在新鲜的MIR中[推理变量](./appendix/glossary.html)。
-   接下来，我们执行一些[数据流分析](./appendix/background.html#dataflow)计算移动的数据和时间。
-   然后我们做一个[第二类检查](borrow_check/type_check.html)跨越MIR：此类型检查的目的是确定不同区域之间的所有约束。
-   接下来，我们做[区域推断](borrow_check/region_inference.html)，计算每个区域的值 - 基本上，控制流图中的点，其中每个生命周期必须根据我们收集的约束有效。
-   此时，我们可以计算每个点的“借用范围”。
-   最后，我们再次浏览MIR，查看它所执行的操作并报告错误。例如，如果我们看到类似的声明`*a + 1`，然后我们会检查变量`a`被初始化并且它没有被可变地借用，因为其中任何一个都需要报告错误。执行此检查需要所有先前分析的结果。

[`replace_regions_in_mir`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/fn.replace_regions_in_mir.html
