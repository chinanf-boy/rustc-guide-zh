# 稳定要求

一旦，一个不稳定的功能得到了良好的测试，没有突兀，任何人都可以推动它的稳定。它涉及以下步骤。

- 文档化 PR
- 写一份稳定报告
- FCP
- 稳定 pr

## 文档化 PR

<a name="updating-documentation"></a>

如果存在关于此功能的任何文档，那它应该先在[`Unstable Book`]，位于[`src/doc/unstable-book`]。 如果 PR 稳定，则应删除该功能的页面。

如果 PR 存在文档，则需要将其集成到现有文档中。

如果那里没有文档，就需要添加到 PR。

可能要的，文档更新位置：

- [参考书][the reference]：必须详细更新。
- [The Book][the book]：这可能需要更新，也可能不需要更新，具体取决于。如果您不确定，请在此存储库中打开一个问题，可以对其进行讨论。
- 标准图书馆文档：根据需要。语言功能通常不需要这样做，但如果这是一个功能，它会改变编写示例的方式，例如，当`?`添加到语言中，更新示例很重要。
- [Rust by Example][rust by example]：根据需要。

根据上述提到的，准备更新 PR 的的新功能文档。在整个稳定过程完成之前，这些存储库的维护人员，将保持这些 prs 的打开状态。同时，我们继续下一步。

## 写一份稳定报告

找到该功能的跟踪问题，并创建一个简短的稳定报告。从本质上讲，这将是对功能的简要总结，加上一些到测试用例的链接，显示它按预期工作，以及一个出现和考虑的边缘用例列表。这是我们在稳定之前，所做的最低限度的“尽职调查”。

报告应包括：

- 摘要，显示此功能启用的示例（例如代码片段）。
- 链接到我们的测试套件中，有关该功能的测试用例，并描述该功能在遇到边缘用例时的行为。
- 链接到文档（我们在前面的步骤中，创建的 PR）。
- 任何其他相关信息（此类报告的示例见 Rust Lang/Rust 44494 和 Rust Lang/Rust 28237）。
- 如果稳定是针对 RFC 的，则加上任何未解决问题的解决方案。

## FCP

如果负责跟踪此功能的团队，其中某些成员同意稳定此功能，他们将通过讨论，启动 FCP（最终讨论期）过程。

```bash
@rfcbot fcp merge
```

其他团队成员将审查该提案。如果最终的决定是要稳定下来，我们将继续进行实际的代码修改。

## 稳定 pr

一旦我们决定稳定一个功能，我们就需要有一个 pr，它可以真正实现稳定功能。这些类型的 prs ，是一个参与 Rust 的很好方法，因为它们带您浏览源代码。

这是如何稳定一个功能的通用指南 —— 当然，每个功能都是不同的，因此一些功能可能需要超出本指南所说的步骤。

注意：在稳定任何功能之前，它应该出现在文档中。

### 更新，feature_gate 的列表

在[`src/libsyntax/feature_gate.rs`]，有关于功能的中心列表。对`declare_features!`宏搜索。您要稳定的功能应该有一个条目，比如（这个例子取自[rust-lang/rust#32409]:

```rust,ignore
// pub(restricted) visibilities (RFC 1422)
(active, pub_restricted, "1.9.0", Some(32409)),
```

上面的代码行，会向下移动到 "已接纳(accepted)"
功能列表, accepted 列表会单独调用了`declare_features!`。
当功能准备好了，会如下所示:

```rust,ignore
// pub(restricted) visibilities (RFC 1422)
(accepted, pub_restricted, "1.31.0", Some(32409)),
// note that we changed this
```

请注意，版本号将更新为稳定版本的版本号，表明此功能出现的版本。这可以通过咨询[the forge](https://forge.rust-lang.org/)找到，这将指导您下一个稳定的版本号。因为今天的代码，将在那个日期进入测试版，然后在下个日期，变成稳定版。所以，我加了 1。如在撰写本文时，下一个稳定版本（即当前的 beta 版本）是 1.30.0，因此我在上面写了 1.31.0(也就是 nightly 版本，也是新功能最先且应该出现的版本)。

### 删除 feature_gate 的现有用法

在代码库中，搜索下一次的功能字符串（在本例中，`pub_restricted`），找到它出现的位置。改变`#![feature(XXX)]`用法，主要从`libstd`和任何 Rust 的箱中，改为`#![cfg_attr(stage0, feature(XXX))]`。仅包括 Stage0 的 feature-gate，它是使用当前 beta 构建的（这是必需的，因为该功能在当前 beta 中，仍然不稳定）。

另外，从任何测试中，删除这些字符串。如果有专门针对 feature-gate 的测试（即，测试 feature-gate 需要使用功能，但没有其他测试），只需删除测试。

### 不用要求 feature-gate ，才能使用功能

最重要的是，如果 feature-gate 中不存在，则删除标记错误的代码（因为，功能现在被认为是稳定的）。如果该功能使用了一些新的语法，是能够检测到它的，这种代码的常见位置是同个`feature_gate.rs`文件。例如，您可能会看到如下代码：

```rust,ignore
gate_feature_post!(&self, pub_restricted, span,
 "`pub(restricted)` syntax is experimental");
```

如果`pub_restricted`功能未启用，这个`gate_feature_post!`宏会打印错误。现在不需要了，因`#[pub_restricted]`是稳定的。

对于功能的更细微处理，您可以找到如下代码：

```rust,ignore
if self.tcx.sess.features.borrow().pub_restricted { /* XXX */ }
```

如果不存在功能标志，这个`pub_restricted`字段（显然以功能命名）通常为 false；如果存在，则为 true。因此，把代码转换为，假定字段为 true。就能做到不少事情。

```rust,ignore
if self.tcx.sess.features.borrow().pub_restricted { /* XXX */ }
becomes
/* XXX */

if self.tcx.sess.features.borrow().pub_restricted && something { /* XXX */ }
 becomes
if something { /* XXX */ }
```

[rust-lang/rust#32409]: https://github.com/rust-lang/rust/issues/32409
[`src/libsyntax/feature_gate.rs`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/feature_gate/index.html
[the reference]: https://github.com/rust-lang-nursery/reference
[the book]: https://github.com/rust-lang/book
[rust by example]: https://github.com/rust-lang/rust-by-example
[`unstable book`]: https://doc.rust-lang.org/unstable-book/index.html
[`src/doc/unstable-book`]: https://github.com/rust-lang/rust/tree/master/src/doc/unstable-book
