# 实现新功能

当您想要在编译器中实现一个新的重要特性时，您需要通过这个过程来确保一切顺利进行。

## @rfcbot（p）fcp流程

当变更很小且无争议时，只需编写一个pr，并从知道代码这一部分的人那里获取r+，就可以完成。但是，如果这一变化可能会引起争议，那么在没有团队其他成员一致同意的情况下推动它将是一个坏主意（从“分布式系统”的角度来看，以确保你不会破坏任何你不知道的东西，从社会的角度来看，以避免公关冲突）。

如果这样的变更看起来太小而不需要一个完整的正式的RFC过程（例如代码的大重构，或者“技术上突破性的”变更，或者基本上相当于一个小特性的“大错误修复”），但是仍然太有争议或太大而无法通过一个R+，您可以启动一个pfcp（或者，如果您没有R+权限，可以问一个拥有让他们开始一个-除非他们自己有顾虑，否则他们应该这样做。

同样，只有在你需要共识的情况下才需要pfcp过程——如果你认为没有人会对你的改变有问题，那么只使用r+就可以了。例如，添加或修改不稳定的命令行标志或属性而不使用pfcp进行编译器开发或标准库使用是可以的，只要您不希望它们在夜间生态系统中被广泛使用。

您不需要让实现完全准备好R+来请求一个pfcp，但是至少要有一个概念证明，这样人们才能看到您所说的内容，这通常是一个好主意。

这就开始了一个“建议的最后评论期”（pfcp），它要求团队的所有成员签署fcp。在他们都这样做之后，有一个长达一周的“最后评论期”，每个人都可以发表评论，如果没有提出新的担忧，公关/问题将得到FCP的批准。

## 写作特点的逻辑

为了以一种有效的方式实现一个特性，您可能需要经历一些“逻辑”环。

### 警告周期

在某些情况下，功能或错误修复可能会在某些边缘情况下破坏某些现有程序。在这种情况下，您可能需要运行一个弹坑来评估影响，并可能添加一个未来的兼容性lint，类似于用于[编辑门控线头](./diag.md#edition-gated-lints).

### 稳定性

我们[重视铁锈的稳定性]. 在稳定状态下工作和运行的代码应该（大部分）不会中断。因此，我们不希望只在团队一致同意和代码审查的情况下向世界发布一个特性——我们希望在夜间获得使用该特性的实际经验，并且我们可能希望基于该经验更改该特性。

考虑到这一点，我们必须确保用户不会意外地依赖于这一新功能——否则，特别是如果实验需要时间或延迟，而这一功能将使火车达到稳定状态，那么它最终将实际上是稳定的，我们将无法在不破坏人们代码的情况下对其进行更改。

我们这样做的方法是确保所有的新特性都是特性门控的—如果没有启用特性门控，它们就不能使用。（`#[feature(foo)]`，这在稳定/测试版编译器中无法实现。见[代码稳定性]技术细节部分。

最后，在我们获得足够的使用该特性的经验、进行必要的更改并得到满足之后，我们使用所描述的稳定过程将其向世界公开。[在这里]. 在此之前，特性并不是一成不变的：特性的每个部分都可以更改，或者特性可能被完全重写或删除。功能不应该因为一年的不稳定和不变而获得使用期。

<a name = "tracking-issue"></a>

### 跟踪问题

为了跟踪不稳定特性的状态，我们在夜间使用它时获得的经验，以及阻止其稳定的问题，每个特性门都需要跟踪问题。

有关该特性的一般性讨论应该在跟踪问题上进行。

对于具有RFC的功能，您应该使用RFC对该功能的跟踪问题。

对于其他功能，您必须对该功能进行跟踪。问题标题应为“跟踪您的功能问题”。

为了跟踪功能的问题（与将来的兼容性警告相反），我认为描述不必包含任何特定的内容。通常，我们使用github列表放置稳定所需的项目列表，例如

```txt
    **Steps:**

    - [ ] Implement the RFC (cc @rust-lang/compiler -- can anyone write
          up mentoring instructions?)
    - [ ] Adjust documentation ([see instructions on forge][doc-guide])
    - Note: no stabilization step here.
```

<a name="stability-in-code"></a>

## 代码稳定性

要实现新的不稳定特性，需要遵循以下步骤：

1.  打开[跟踪问题]-如果您有一个RFC，您可以使用RFC的跟踪问题。

2.  选择功能入口的名称（对于RFC，使用RFC中的名称）。

3.  将功能入口声明添加到`libsyntax/feature_gate.rs`在活动中`declare_features`阻止：

```rust,ignore
    // description of feature
    (active, $feature_name, "$current_nightly_version", Some($tracking_issue_number), $edition)
```

在哪里？`$edition`具有类型`Option<Edition>`，通常只是`None`.

例如：

```rust,ignore
    // allow '|' at beginning of match arms (RFC 1925)
(   active, match_beginning_vert, "1.21.0", Some(44101), None),
```

当前版本实际上并不重要——重要的版本是在稳定某个特性时。

4.  除非设置了功能入口，否则禁止使用新功能。您可以使用表达式在编译器中的大多数地方检查它`tcx.features().$feature_name`（或）`sess.features_untracked().borrow().$feature_name`如果TCX不可用）

    如果未设置特征门，则应维护前特征行为或引发错误，具体取决于什么是有意义的。

5.  通过创建`feature-gate-$feature_name.rs`和`feature-gate-$feature_name.stderr`文件下`src/test/ui/feature-gates`目录。

6.  在不稳定的书中添加一节`src/doc/unstable-book/src/language-features/$feature_name.md`.

7.  为新特性编写大量测试。未经测试的PRS将不被接受！

8.  让你的公关审查并登陆。您现在已经成功地在Rust中实现了一个特性！

[value the stability of rust]: https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md

[stability in code]: #stability-in-code

[here]: https://rust-lang.github.io/rustc-guide/stabilization_guide.html

[tracking issue]: #tracking-issue
