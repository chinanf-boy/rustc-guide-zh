# 粉笔概述

> 粉笔正在大量开发中，因此如果这些链接中的任何一个被破坏，或者任何信息与代码不一致或过时，请[打开一个问题][rustc-issues]所以我们可以修复它。如果你能自己解决这个问题，我们会喜欢你的贡献！

[粉笔][chalk]通过将rust代码“降低”到一种逻辑程序中，然后我们就可以对其执行查询，从而明确地从逻辑编程的角度重新构建rust的特征系统。（见[*降低到逻辑*][lowering-to-logic]和[*下沉规则*][lowering-rules]）它的目标是成为一个可执行的，高度可读的生锈特性系统规范。

这项工作有许多预期的好处。它将把我们现有的、有点特别的实现整合成更具原则性和表现力的东西，这些东西在角落的情况下应该表现得更好，并且更容易扩展。

## 粉笔结构

粉笔有两个主要的“产品”。第一个是[`chalk_engine`][chalk_engine]板条箱，定义了核心[SLG求解器][slg]. 这是Rustc使用的部件。

剩下的粉笔可以看作是一个精心设计的测试工具。粉笔能够像“程序”那样解析锈迹，将它们降为逻辑，并对它们执行查询。

这是粉笔回复中的一个例子，粉笔。在给它输入程序之后，我们对它执行一些查询。

```rust,ignore
?- program
Enter a program; press Ctrl-D when finished
| struct Foo { }
| struct Bar { }
| struct Vec<T> { }
| trait Clone { }
| impl<T> Clone for Vec<T> where T: Clone { }
| impl Clone for Foo { }

?- Vec<Foo>: Clone
Unique; substitution [], lifetime constraints []

?- Vec<Bar>: Clone
No possible solution.

?- exists<T> { Vec<T>: Clone }
Ambiguous; no inference guidance
```

您可以在[单元测试][chalk-test-example].

接下来，我们将完成产生上述输出所需的每个阶段。

### 句法分析（句法分析）[查克分析_]）

粉笔被设计成与Rust编译器结合在一起，因此它处理的语法和概念大量借用Rust。为了便于测试，可以单独运行粉笔，因此粉笔包含了一个类似于rust的语法分析器。这个语法与rust ast和语法是正交的。它并不打算看起来完全相同，也不支持完全相同的语法。

解析器采用该语法并生成一个[抽象语法树（ast）][ast]. 你可以找到[AST的完整定义][chalk-ast]在源代码中。

语法包含了一些我们所知道和喜爱的生锈的东西，例如：特性、内爆和结构定义。解析通常是程序进行转换的第一个“阶段”，以便成为粉笔可以理解的格式。

### 铁锈中间体表示（[生锈_]）

在得到AST之后，我们将其转换为更方便的中间表示，称为[`rust_ir`][rust_ir].这有点类似于[希尔]生锈了。转换为红外的过程称为*降低*.

这个[`rust_ir::Program`][rust_ir-program]结构包含一些“生锈的东西”，但以不同的方式索引和访问。例如，如果您有`Foo<Bar>`，我们将代表`Foo`在AST中作为字符串，但在`rust_ir::Program`，我们使用数字索引（`ItemId`）。

这个[红外源代码][ir-code]包含完整的定义。

### 粉笔中间表示法（[粉笔_]）

一旦我们有了锈，是时候把它转换成“程序条款”。A[`ProgramClause`]基本上是以下之一：

-   A [条款]形式的`consequence :- conditions`在哪里？`:-`读作“如果”和`conditions = cond1 && cond2 && ...`
-   形式的一个普遍量化的从句`forall<T> { consequence :- conditions }`
    -   `forall<T> { ... }`用于表示[通用量化]. 请参见[降低到逻辑][lowering-forall]更多信息。
    -   需要注意的关键问题`forall`就是我们不允许你“量化”特性，只允许类型和区域（生命周期）。也就是说，你不能制定这样的规则`forall<Trait> { u32: Trait }`那会说`u32`实现所有特征”。但是你可以说`forall<T> { T: Trait }`“意义”`Trait`由所有类型实现。
    -   `forall<T> { ... }`在代码中使用[`Binders<T>`结构][binders-struct].

*参见：[目标和条款][goals-and-clauses]*

这就是我们把特征系统的规则编码成逻辑的地方。例如，如果我们有以下锈迹：

```rust,ignore
impl<T: Clone> Clone for Vec<T> {}
```

我们生成以下程序子句：

```rust,ignore
forall<T> { (Vec<T>: Clone) :- (T: Clone) }
```

这条规则规定`Vec<T>: Clone`只有满足于`T: Clone`也满足（即“可证明”）。

类似[`rust_ir::Program`][rust_ir-program]粉笔笔画的定义是“像铁锈一样的东西”[`ProgramEnvironment`]这是“纯粹的逻辑”。该结构中的主字段是`program_clauses`，其中包含[`ProgramClause`]由规则模块生成。

#### 规则

这个`rules`模块（[源代码][rules-src]）定义我们在rust ir中为每个项使用的逻辑规则。它通过迭代每一个特性、impl等并发出来自每个特性的规则来工作。

*参见：[下沉规则][lowering-rules]*

#### 井然有序的检查

作为降低逻辑的一部分，我们还进行了一些“良好形式”检查。见[`rules::wf`源代码][rules-wf-src]因为这些都是在哪里完成的。

*参见：[形态检查][wf-checking]*

#### 一致性

函数`record_specialization_priorities`在`coherence`模块（[源代码][coherence-src]）检查“一致性”，这意味着它可以确保同一类型的两个具有相同特征的impl不存在。

### 求解器[查尔基解_]）

最后，当我们收集了所有我们关心的程序子句后，我们希望对其执行查询。找到这些查询的答案的组件称为*求解器*.

*参见：[SLG求解器][slg]*

## 板条箱

粉笔的功能被分解成以下板条箱：

-   [**查尔基发动机**][chalk_engine]：定义核心[SLG求解器][slg].
-   [**查尔基尔**][chalk_ir]：定义粉笔的类型、生命周期和目标的内部表示。
-   [**查尔基解**][chalk_solve]组合`chalk_ir`和`chalk_engine`，有效地。
    -   [`chalk_engine::context`][engine-context]提供必要的挂钩。
-   [**查克分析**][chalk_parse]：定义原始AST和分析器。
-   [**粉笔**][doc-chalk]把所有的东西都聚在一起。定义以下模块：
    -   [`rust_ir`][rust_ir]包含ast的“hir-like”形式
        -   `rust_ir::lowering`，它将ast转换为`rust_ir`
    -   `rules`，实现逻辑规则转换`rust_ir`到`chalk_ir`
    -   `coherence`，实现一致性规则
    -   还包括[查基][chalki]白垩

[在GitHub上浏览源代码](https://github.com/rust-lang-nursery/chalk)

## 测试

Chalk有一个测试框架，用于将程序降低到逻辑，检查降低的逻辑，并对其执行查询。这就是我们如何测试粉笔本身的实现，以及[下沉规则][lowering-rules].

粉笔测试的主要类型是**目标测试**. 它们包含一个程序（预期将成功降低到逻辑），以及一组查询（目标）以及预期的输出。这里有一个[例子][chalk-test-example]. 由于粉笔的输出可能很长，所以目标测试只支持指定输出的前缀。

**降低试验**检查在向解算器发出查询之前发生的阶段：[降低至生锈][chalk-test-lowering]和[井然有序的检查][chalk-test-wf]在那之后发生的。

### 内部测试

目标测试使用[`test!`宏][test-macro]这需要粉笔的锈样语法，并通过上面描述的完整管道运行。宏最终调用[`solve_goal`功能][solve_goal].

同样，降低测试使用[`lowering_success!`和`lowering_error!`宏指令][test-lowering-macros].

## 更多资源

-   [粉笔源代码](https://github.com/rust-lang-nursery/chalk)
-   [粉笔词汇表](https://github.com/rust-lang-nursery/chalk/blob/master/GLOSSARY.md)

### 博客帖子

-   [降低锈病的逻辑特征](http://smallcultfollowing.com/babysteps/blog/2017/01/26/lowering-rust-traits-to-logic/)
-   [粉笔统一，第一部分](http://smallcultfollowing.com/babysteps/blog/2017/03/25/unification-in-chalk-part-1/)
-   [粉笔统一，第2部分](http://smallcultfollowing.com/babysteps/blog/2017/04/23/unification-in-chalk-part-2/)
-   [粉笔中的否定推理](http://aturon.github.io/blog/2017/04/24/negative-chalk/)
-   [粉笔中的查询结构](http://smallcultfollowing.com/babysteps/blog/2017/05/25/query-structure-in-chalk/)
-   [粉笔中的循环查询](http://smallcultfollowing.com/babysteps/blog/2017/09/12/tabling-handling-cyclic-queries-in-chalk/)
-   [粉笔的按需SLG解算器](http://smallcultfollowing.com/babysteps/blog/2018/01/31/an-on-demand-slg-solver-for-chalk/)

[goals-and-clauses]: ./goals-and-clauses.html

[hir]: ../hir.html

[lowering-forall]: ./lowering-to-logic.html#type-checking-generic-functions-beyond-horn-clauses

[lowering-rules]: ./lowering-rules.html

[lowering-to-logic]: ./lowering-to-logic.html

[slg]: ./slg.html

[wf-checking]: ./wf.html

[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree

[chalk]: https://github.com/rust-lang-nursery/chalk

[rustc-issues]: https://github.com/rust-lang-nursery/rustc-guide/issues

[universal quantification]: https://en.wikipedia.org/wiki/Universal_quantification

[`programclause`]: https://rust-lang-nursery.github.io/chalk/doc/chalk_ir/enum.ProgramClause.html

[`programenvironment`]: https://rust-lang-nursery.github.io/chalk/doc/chalk_ir/struct.ProgramEnvironment.html

[chalk_engine]: https://rust-lang-nursery.github.io/chalk/doc/chalk_engine/index.html

[chalk_ir]: https://rust-lang-nursery.github.io/chalk/doc/chalk_ir/index.html

[chalk_parse]: https://rust-lang-nursery.github.io/chalk/doc/chalk_parse/index.html

[chalk_solve]: https://rust-lang-nursery.github.io/chalk/doc/chalk_solve/index.html

[doc-chalk]: https://rust-lang-nursery.github.io/chalk/doc/chalk/index.html

[engine-context]: https://rust-lang-nursery.github.io/chalk/doc/chalk_engine/context/index.html

[rust_ir-program]: https://rust-lang-nursery.github.io/chalk/doc/chalk/rust_ir/struct.Program.html

[rust_ir]: https://rust-lang-nursery.github.io/chalk/doc/chalk/rust_ir/index.html

[binders-struct]: https://github.com/rust-lang-nursery/chalk/blob/94a1941a021842a5fcb35cd043145c8faae59f08/src/ir.rs#L661

[chalk-ast]: https://github.com/rust-lang-nursery/chalk/blob/master/chalk-parse/src/ast.rs

[chalk-test-example]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/test.rs#L115

[chalk-test-lowering-example]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/rust_ir/lowering/test.rs#L8-L31

[chalk-test-lowering]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/rust_ir/lowering/test.rs

[chalk-test-wf]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/rules/wf/test.rs#L1

[chalki]: https://rust-lang-nursery.github.io/chalk/doc/chalki/index.html

[clause]: https://github.com/rust-lang-nursery/chalk/blob/master/GLOSSARY.md#clause

[coherence-src]: https://github.com/rust-lang-nursery/chalk/blob/master/src/coherence.rs

[ir-code]: https://github.com/rust-lang-nursery/chalk/blob/master/src/rust_ir.rs

[rules-environment]: https://github.com/rust-lang-nursery/chalk/blob/94a1941a021842a5fcb35cd043145c8faae59f08/src/rules.rs#L9

[rules-src]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/rules.rs

[rules-wf-src]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/rules/wf.rs

[solve_goal]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/test.rs#L85

[test-lowering-macros]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/test_util.rs#L21-L54

[test-macro]: https://github.com/rust-lang-nursery/chalk/blob/4bce000801de31bf45c02f742a5fce335c9f035f/src/test.rs#L33
