# 实现新功能

当您想要在编译器中实现一个新的重要功能时，您需要经过这个过程来确保一切顺利进行。

## 这个 @rfcbot (p)FCP 流程

当变更很小且无争议时，只需编写一个 PR，并从知道这部分代码的人那里获取 r+权限，就可以完成。但是，如果这一变化会引起争议，那么在没有团队其他成员一致同意的情况下，推动它将是一个坏主意（从“分布式系统”的角度来看，确保你不会破坏任何你不知道的东西，从社会的角度来看，以避免 PR 冲突）。

如果有个变更看起来太小，不需要一个完整的正式的 RFC 过程（例如代码的大重构，或者“技术上突破性的”变更，或者基本上相当于一个小功能的“大错误修复”），但是仍然有争议或太重要，而无法获得一个 r+，您可以启动一个 pFCP（或者，如果您没有 r+权限，可以问一个有的人，让他们开始一个 —— 除非他们自己有顾虑，否则他们应该会帮忙。

同样，只有在需要达成共识的情况下，才需要 pFCP 过程 —— 如果你认为没有人会对你的改变有问题，那么只使用 r+就可以了。例如，想要添加或修改不稳定的命令行标志或属性，而不进行 pFCP ，就编译器开发或标准库使用是可以的，只要您不打算它们在夜间生态系统中，被广泛使用。

您不需要让实现完全准备好，才 r+，或是请求一个 pFCP，但是至少要有一个概念证明，这样人们才能看到您所说的内容，这通常是一个好主意。

要开始了一个“提议的最后评论期”（pFCP），需要团队的所有成员签署 这份 FCP。在他们都这样做之后，有一个长达一周的“最后评论期”，每个人都可以发表评论，如果没有提出新的担忧，PR/issue 将得到 FCP 批准。

## 编写功能的逻辑

为了以一种有效的方式实现一个功能，您可能需要经历一些“逻辑”环节。

### 警告周期

在某些情况下，一个功能或错误修复可能会在某些边缘情况下，破坏某些现有程序。在这种情况下，您可能需要运行一个 crater 来评估影响，并可能添加一个具备功能兼容性的 lint，类似于[edition-gated lints](./diag.zh.md#edition-gated-lints)的使用。

### 稳定性

我们[重视铁锈的稳定性][value the stability of rust]。维持稳定状态下，工作和运行的代码（大部分）不出故障。因此，我们不希望只在团队一致同意和代码审查的情况下，向世界发布一个功能 —— 我们希望在夜间版本，先获得使用该功能的实际经验，然后根据经验更改该功能。

考虑到这一点，我们必须确保用户不会，意外地依赖于这一新功能 —— 否则的话，特别是，若实验需要时间或延迟，而这一功能要经过一段旅程，才能达到稳定的情况下，即便实际上最终是稳定的，但是我们也无法在不破坏人们代码的情况下，对其进行更改。

确认所有新功能的方法，就是 feature
gated — 如果没有启用 feature
gate，它们就不能使用（`#[feature(foo)]`，这在稳定/测试版编译器中无法实现。见[代码稳定性][stability in code]的技术细节部分。

最后，在我们获得足够多，使用该功能的经验、进行必要的更改并得到满足之后，我们使用所描述的[这里]的稳定过程，将其向世界公开。 在此之前，功能并不是一成不变的：功能的每个部分都可以更改，或者功能可能被完全重写或删除。若是有功能在一年都处于不稳定和不变的状态，那就应该考虑结束使用。

<a name = "tracking-issue"></a>

### 跟踪问题

为了跟踪不稳定功能的状态，我们会在夜间使用它时，以获得经验，以及弄清阻止它稳定的问题。每个 feature-gate 都需要跟踪问题。

有关该功能的一般讨论，应该在跟踪问题上进行。

对于具有 RFC 的功能，您应该使用 RFC 对该功能的跟踪问题。

对于其他功能，您必须对该功能进行跟踪。问题标题应为“Tracking issue
for YOUR FEATURE”。

为了跟踪功能的问题（与功能-兼容性警告相反），我认为描述不必包含任何特定的内容。通常，我们使用 github ，放置稳定过程所需项的列表，例如

```txt
    **Steps:**

    - [ ] Implement the RFC (cc @rust-lang/compiler -- can anyone write
          up mentoring instructions?)
    - [ ] Adjust documentation ([see instructions on forge][doc-guide])
    - Note: no stabilization step here.
```

<a name="stability-in-code"></a>

## 代码稳定性

要实现新的不稳定功能，需要遵循以下步骤：

1.  打开一个[跟踪问题][tracking issue]-如果您有一个 RFC，您可以使用 RFC 的跟踪问题。

2.  选择 feature gate 的名称（若是 RFC，则使用 RFC 中的名称）。

3.  将 feature gate 声明在`libsyntax/feature_gate.rs`，添加到 active 的`declare_features`代码块：

```rust,ignore
    // description of feature
    (active, $feature_name, "$current_nightly_version", Some($tracking_issue_number), $edition)
```

这里的`$edition`具有类型`Option<Edition>`，通常只是`None`.

例如：

```rust,ignore
    // allow '|' at beginning of match arms (RFC 1925)
(   active, match_beginning_vert, "1.21.0", Some(44101), None),
```

当前版本实际上并不重要 —— 重要的是，某个功能稳定时的版本。

4.  除非设置了 feature gate ，否则禁止使用新功能。您可以在编译器中的大多数地方，用表达式检查它，如`tcx.features().$feature_name`（或，如果 tcx 不可用`sess.features_untracked().borrow().$feature_name`）

    如果未设置 feature gate，则应维护功能之前的行为，或其引发的错误，具体取决于什么才是有意义的。

5.  添加一个测试，确保一个不具备 feature gate 的功能，是不能用的。具体在`src/test/ui/feature-gates`目录下，创建`feature-gate-$feature_name.rs`和`feature-gate-$feature_name.stderr`文件。

6.  在不稳定的书(unstable book)中添加一节，如`src/doc/unstable-book/src/language-features/$feature_name.md`.

7.  为新功能编写大量测试。未经测试的 PR 将不被接受！

8.  让你的 PR 审查员登记下。您现在已经成功地在 Rust 中实现了一个功能！

[value the stability of rust]: https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md
[stability in code]: #stability-in-code
[here]: https://rust-lang.github.io/rustc-guide/stabilization_guide.html
[tracking issue]: #tracking-issue
