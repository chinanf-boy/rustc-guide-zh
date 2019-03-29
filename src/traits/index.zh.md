# trait 解决（新式）

> 🚧 本章描述了 trait 解决的“新式”方法。这还处于[正在实现的过程][wg];本章作为一种正在进行的设计文件。如果您想阅读当前 trait 解算器（solver）的工作原理，请查看[其他章节](./resolution.zh.html)。🚧
>
> 顺便说一句，如果你想帮助 hacking 新解算器，你会找到参与[trait 工作组之问题跟踪][wg]的说明！

[wg]: https://github.com/rust-lang/rust/issues/48416

新式 trait 解算器，基于[chalk][chalk]完成的工作。Chalk 在逻辑编程方面，明确地重述了 Rust 的 trait 系统。它通过将 Rust 代码“降层（lowering）”为一种逻辑程序来做到，然后我们可以用此，来执行查询。

你可以在[Chalk 概略](./chalk-overview.zh.md)章节，阅读更多关于 chalk 本身的信息。

在 rustc 中，解决 trait 是基于几个关键想法：

- [Lowering 到逻辑](./lowering-to-logic.zh.html)，用标准逻辑术语，表达 Rust trait。
  - 该[goals 和 clauses](./goals-and-clauses.zh.html)章节，描述了我们使用的规则的确切形式，以及[lowering 规则](./lowering-rules.zh.html)，更像以一种参考形式，给出完整的降低规则集。
  - [Lazy normalization](./associated-types.zh.html)，这是我们在确定类型是否相等时，用来容纳相关类型的技术。
  - [Region constraints](./regions.zh.html)，这是在 trait 解决过程中积累的，但大部分被忽略。这意味着 trait 解决有效地忽略了，所涉及的精确区域 - 但我们仍然记得对它们的约束，以便可以通过类型检查器，检查这些约束。
- [Canonical queries](./canonical-queries.zh.html)，这让我们能够解决 trait 问题（比如“`Foo`是否为`Bar`类型实现了？“）一次，然后在许多不同的推断上下文中，独立地应用相同的结果。

> 这不是一个完整的主题列表。有关更多信息，请参阅侧栏。

## 正在进行的工作

新型 trait 解决方案的设计，目前在两个地方进行：

**chalk**。该[chalk][chalk]存储库，是我们尝试 trait 系统的新想法和设计的地方。它主要由两部分组成：

- 单元测试框架，对新式 trait 系统定义的逻辑规则，关注其正确性和可行性。
- 该[`chalk_engine`][chalk_engine]箱子，它定义了单元测试框架和 rustc 中，使用的新式 trait 解算器。

**rustc**。一旦我们对逻辑规则感到满意，我们就会继续在 rustc 中实现它们。这主要发生在[`librustc_traits`][librustc_traits]。

[chalk]: https://github.com/rust-lang-nursery/chalk
[chalk_engine]: https://github.com/rust-lang-nursery/chalk/tree/master/chalk-engine
[librustc_traits]: https://github.com/rust-lang/rust/tree/master/src/librustc_traits
