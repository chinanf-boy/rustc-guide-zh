# 特征解决(新型)

> solvin🚧本章描述"new-style"特征[实施的过程][wg];[本章作为一种正在进行des](./resolution.html)。
>
> 🚧[品质工作小组跟踪问题][wg]!

[wg]: https://github.com/rust-lang/rust/issues/48416

新型特征解算器是基于工作做[粉笔][chalk]。

粉笔重铸系统明确生锈的特征[粉笔的概述](./chalk-overview.md)部分。

特征解决rustc是基于一些关键的我

-   [降低对逻辑](./lowering-to-logic.html),表达了锈特征的标准
    -   的[目标和条款](./goals-and-clauses.html)章描述我们我们规则的精确形式[降低规则](./lowering-rules.html)给出了完整的铁道部降低规则
    -   [懒惰的正常化](./associated-types.html),这是我们使用的技术,以适应的屁股
    -   [地区限制](./regions.html)解决期间积累的特质,但m
-   [标准查询](./canonical-queries.html)让我们解决品质问题(如“`Foo`实现的类型`Bar`?”)一次,然后应用相同的结果能自食其力

> 这不是一个主题的完整列表。

## 看到sid

解决currentl新型的设计特点

**粉笔**。[的][chalk]库是我们尝试新的想法

-   一个单元测试框架的正确性和f
-   的[`chalk_engine`][chalk_engine]箱,它定义了新型特征解算器

**rustc**。[`librustc_traits`][librustc_traits]一旦我们满意的逻辑规则,我们专业

[chalk]: https://github.com/rust-lang-nursery/chalk

[chalk_engine]: https://github.com/rust-lang-nursery/chalk/tree/master/chalk-engine

[librustc_traits]: https://github.com/rust-lang/rust/tree/master/src/librustc_traits
