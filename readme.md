# rust-lang/rustc-guide [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 构建一个，解释 rustc 如何工作的指南 」

[中文](readme.md) | [english](https://github.com/rust-lang/rustc-guide)

---

## 校对 🀄️

<!-- doc-templite START generated -->
<!-- repo = 'rust-lang/rustc-guide' -->
<!-- commit = '0456aaa9e197e6d3f8349bca6299becb836e4070' -->
<!-- time = '2019-03-01' -->

| 翻译的原文 | 与日期        | 最新更新 | 更多                       |
| ---------- | ------------- | -------- | -------------------------- |
| [commit]   | ⏰ 2019-03-01 | ![last]  | [中文翻译][translate-list] |

[last]: https://img.shields.io/github/last-commit/rust-lang/rustc-guide.svg
[commit]: https://github.com/rust-lang/rustc-guide/tree/0456aaa9e197e6d3f8349bca6299becb836e4070

# [摘要](src/SUMMARY.md)

- [x] [关于本指南](src/about-this-guide.zh.md)

---

- [x] [第 1 部分：构建，调试和帮助 Rustc](src/part-1-intro.zh.md)
- [x] [关于编译器团队](src/compiler-team.zh.md)
- [x] [如何构建编译器并运行您构建的内容](src/how-to-build-and-run.zh.md)
  - [x] [构建和安装分发工件](src/build-install-distribution-artifacts.zh.md)
  - [x] [文档化编译器](src/compiler-documenting.zh.md)
- [x] [编译器的测试框架](src/tests/intro.zh.md)
  - [x] [运行测试](src/tests/running.zh.md)
  - [x] [添加新测试](src/tests/adding.zh.md)
  - [x] [运用`compiletest`+ 头命令 来控制测试执行](src/compiletest.zh.md)
- [x] [演练：贡献示例](src/walkthrough.zh.md)
- [x] [实现新功能](src/implementing_new_features.zh.md)
- [x] [功能的稳定过程](src/stabilization_guide.zh.md)
- [ ] [调试编译器](src/compiler-debugging.zh.md)
- [x] [分析编译器](src/profiling.zh.md)
  - [ ] [用 linux perf 工具](src/profiling/with_perf.zh.md)
- [ ] [编码惯例](src/conventions.zh.md)

---

- [ ] [第 2 部分：rustc 如何工作](src/part-2-intro.zh.md)
- [ ] [编译器源的高级概述](src/high-level-overview.zh.md)
- [ ] [Rustc 驱动程序](src/rustc-driver.zh.md)
  - [ ] [Rustdoc](src/rustdoc.zh.md)
- [ ] [查询：需求驱动的编译](src/query.zh.md)
  - [ ] [详细查询评估模型](src/queries/query-evaluation-model-in-detail.zh.md)
  - [ ] [增量编译](src/queries/incremental-compilation.zh.md)
  - [ ] [增量编译详细信息](src/queries/incremental-compilation-in-detail.zh.md)
  - [ ] [调试和测试](src/incrcomp-debugging.zh.md)
- [ ] [解析器](src/the-parser.zh.md)
- [ ] [`#[test]`履行](src/test-implementation.zh.md)
- [ ] [宏观扩张](src/macro-expansion.zh.md)
- [ ] [名称解析](src/name-resolution.zh.md)
- [ ] [HIR（高级 IR）](src/hir.zh.md)
  - [ ] [将 AST 降低到 HIR](src/lowering.zh.md)
  - [ ] [调试](src/hir-debugging.zh.md)
- [ ] [该`ty`module：表示类型](src/ty.zh.md)
- [ ] [种](src/kinds.zh.md)
- [ ] [类型推断](src/type-inference.zh.md)
- [ ] [特质解决（旧式）](src/traits/resolution.zh.md)
  - [ ] [排名较高的特质界限](src/traits/hrtb.zh.md)
  - [ ] [缓存细微之处](src/traits/caching.zh.md)
  - [ ] [专业化](src/traits/specialization.zh.md)
- [ ] [特质解决（新式）](src/traits/index.zh.md)
  - [ ] [降低到逻辑](src/traits/lowering-to-logic.zh.md)
    - [ ] [目标和条款](src/traits/goals-and-clauses.zh.md)
    - [ ] [平等和相关类型](src/traits/associated-types.zh.md)
    - [ ] [隐含的界限](src/traits/implied-bounds.zh.md)
    - [ ] [区域限制](src/traits/regions.zh.md)
    - [ ] [降低模块在 rustc](src/traits/lowering-module.zh.md)
    - [ ] [降低规则](src/traits/lowering-rules.zh.md)
    - [ ] [良好的形成检查](src/traits/wf.zh.md)
  - [ ] [规范查询](src/traits/canonical-queries.zh.md)
    - [ ] [规范化](src/traits/canonicalization.zh.md)
  - [ ] [SLG 求解器](src/traits/slg.zh.md)
  - [ ] [粉笔概述](src/traits/chalk-overview.zh.md)
  - [ ] [参考书目](src/traits/bibliography.zh.md)
- [ ] [类型检查](src/type-checking.zh.md)
  - [ ] [方法查找](src/method-lookup.zh.md)
  - [ ] [方差](src/variance.zh.md)
  - [ ] [存在类型](src/existential-types.zh.md)
- [ ] [MIR（中级 IR）](src/mir/index.zh.md)
  - [ ] [MIR 建设](src/mir/construction.zh.md)
  - [ ] [MIR 访客和遍历](src/mir/visitor.zh.md)
  - [ ] [MIR 通过：获取功能的 MIR](src/mir/passes.zh.md)
  - [ ] [MIR 优化](src/mir/optimizations.zh.md)
  - [ ] [调试](src/mir/debugging.zh.md)
- [ ] [借用检查员](src/borrow_check.zh.md)
  - [ ] [跟踪移动和初始化](src/borrow_check/moves_and_initialization.zh.md)
    - [ ] [移动路径](src/borrow_check/moves_and_initialization/move_paths.zh.md)
  - [ ] [MIR 型检查器](src/borrow_check/type_check.zh.md)
  - [ ] [区域推断](src/borrow_check/region_inference.zh.md)
- [ ] [不断评估](src/const-eval.zh.md)
  - [ ] [miri const 评估员](src/miri.zh.md)
- [ ] [参数环境](src/param_env.zh.md)
- [ ] [代码生成](src/codegen.zh.md)
  - [ ] [更新 LLVM](src/codegen/updating-llvm.zh.md)
  - [ ] [调试 LLVM](src/codegen/debugging.zh.md)
- [ ] [发出诊断](src/diag.zh.md)

---

- [ ] [附录 A：愚蠢的统计数据](src/appendix/stupid-stats.zh.md)
- [ ] [附录 B：背景材料](src/appendix/background.zh.md)
- [ ] [附录 C：术语表](src/appendix/glossary.zh.md)
- [ ] [附录 D：代码索引](src/appendix/code-index.zh.md)
- [ ] [src/important-links.zh.md]

<!-- doc-templite END generated -->

### 贡献

欢迎 👏 勘误/校对/更新贡献 😊 [具体贡献请看](https://github.com/chinanf-boy/chinese-translate-list#贡献)

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

这是一项协作工作，旨在构建一个，解释 rustc 如何工作的指南。本指南的目的是帮助新贡献者认清 rustc，以及帮助更有经验的人找出他们之前没有使用过的一些新部分编译器。

[你可在该网页阅读最新版本的指南.](https://rust-lang-nursery.github.io/rustc-guide/)

你也可以找到[编译器本身][rustdocs]的 rustdocs ，这会有用。

[rustdocs]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/

### 为指南做贡献

该指南今天可用了，但还有很多工作要做。

如果您想帮助改进指南，我们欢迎！你可以找到很多问题[issue
tracker](https://github.com/rust-lang/rustc-guide/issues)。只需发表您想要解决的问题的评论，以确保我们不会意外地重复工作。如果您认为缺少某些内容，请打开相关问题！

**一般来说，如果你不知道编译器是如何工作的，那不是问题！**在这种情况下，我们要做的是，安排一些时间给你说明谁**是**知道代码，或谁与你配对，并帮你弄清楚。然后你可以写出你学到的东西。

一般来说，在编写编译器代码的特定部分时，我们建议您链接到编译器的[rustc rustdocs][rustdocs]相关部分。

为了防止意外引入断开的链接，我们使用了`mdbook-linkcheck`。如果安装在您的机器上`mdbook`，将自动调用此链接检查器，否则它将发出警告说无法找到它。

```bash
> cargo install mdbook-linkcheck
```

你会需要`mdbook`版本`>= 0.2`。`linkcheck`您将在`mdbook build`运行时，自动运行。
