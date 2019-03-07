# 稳定要求

一旦一个不稳定的特性得到了良好的测试，没有任何突出的关注，任何人都可以推动它的稳定。它涉及以下步骤。

-   文件PRS
-   写一份稳定报告
-   FCP
-   稳定pr

## 文件PRS

<a name="updating-documentation"></a>

如果存在此功能的任何文档，它应该位于[`Unstable Book`]，位于[`src/doc/unstable-book`]. 如果存在，则应删除功能入口的页面。

如果存在文档，则需要将其集成到现有文档中。

如果那里没有文档，就需要添加。

可能需要更新文档的位置：

-   [参考文献]：必须详细更新。
-   [书]：这可能需要更新，也可能不需要更新，具体取决于。如果您不确定，请在此存储库中打开一个问题，可以对其进行讨论。
-   标准图书馆文件：根据需要。语言特性通常不需要这样做，但如果这是一个特性，它会改变编写示例的方式，例如何时`?`添加到语言中，更新示例很重要。
-   [锈蚀实例]：根据需要。

准备PRS以更新涉及上述存储库新功能的文档。在整个稳定过程完成之前，这些存储库的维护人员将保持这些prs的打开状态。同时，我们可以继续下一步。

## 写一份稳定报告

找到特征的跟踪问题，并创建一个简短的稳定报告。从本质上讲，这将是对特性的简要总结，加上一些到测试用例的链接，显示它按预期工作，以及一个出现和考虑的边缘用例列表。这是我们在稳定之前所做的最低限度的“尽职调查”。

报告应包括：

-   摘要，显示此功能启用的示例（例如代码段）。
-   链接到我们的测试套件中有关该特性的测试用例，并描述该特性在遇到边缘用例时的行为。
-   链接到文档（我们在前面的步骤中创建的prs）。
-   任何其他相关信息（此类报告的示例见Rust Lang/Rust 44494和Rust Lang/Rust 28237）。
-   如果稳定是针对RFC的，则任何未解决问题的解决方案。

## FCP

如果负责跟踪此功能的团队中的任何成员同意稳定此功能，他们将通过评论启动FCP（最终评论期）过程。

```bash
@rfcbot fcp merge
```

其他团队成员将审查该提案。如果最终的决定是要稳定下来，我们将继续进行实际的代码修改。

## 稳定pr

一旦我们决定稳定一个特性，我们就需要有一个pr，它可以真正实现稳定。这些类型的prs是一个很好的方法来参与生锈，因为它们带您浏览一下源代码。

下面是如何稳定一个特性的一般指南——当然，每个特性都是不同的，因此一些特性可能需要超出本指南所说的步骤。

注意：在稳定任何特性之前，它应该出现在文档中。

### 更新功能入口列表

在[`src/libsyntax/feature_gate.rs`]. 搜索`declare_features!`宏。您要稳定的特性应该有一个条目，比如（这个例子取自

[rust-lang/rust#32409]: ```rust,ignore

//发布（受限）可见性（RFC 1422）（活动，发布受限，“1.9.0”，一些（32409）），

````
The above line should be moved down to the area for "accepted"
features, declared below in a separate call to `declare_features!`.
When it is done, it should look like:

```rust,ignore
// pub(restricted) visibilities (RFC 1422)
(accepted, pub_restricted, "1.31.0", Some(32409)),
// note that we changed this
````

请注意，版本号将更新为稳定版本的版本号，此功能将在其中出现。这可以通过咨询找到。[锻造厂](https://forge.rust-lang.org/)，这将指导您下一个稳定的版本号。你想增加1，因为今天登陆的代码将在那个日期进入测试版，然后在那之后变得稳定。所以，在撰写本文时，下一个稳定版本（即当前的beta版本）是1.30.0，因此我在上面写了1.31.0。

### 删除功能入口的现有用途

下一次搜索功能字符串（在本例中，`pub_restricted`）在代码库中找到它出现的位置。改变用途`#![feature(XXX)]`从`libstd`任何生锈的板条箱`#![cfg_attr(stage0, feature(XXX))]`. 这包括仅用于Stage0的特性门，它是使用当前beta构建的（这是必需的，因为该特性在当前beta中仍然不稳定）。

另外，从任何测试中删除这些字符串。如果有专门针对特性门的测试（即，测试特性门需要使用特性，但没有其他测试），只需删除测试。

### 不需要特征门来使用特征

最重要的是，如果特性门不存在，则删除标记错误的代码（因为特性现在被认为是稳定的）。如果由于使用了一些新的语法而可以检测到该特性，则该代码的常见位置是相同的`feature_gate.rs`. 例如，您可能会看到如下代码：

```rust,ignore
gate_feature_post!(&self, pub_restricted, span,
 "`pub(restricted)` syntax is experimental");
```

这个`gate_feature_post!`宏打印错误，如果`pub_restricted`功能未启用。现在不需要了`#[pub_restricted]`是稳定的。

对于更细微的特性，您可以找到如下代码：

```rust,ignore
if self.tcx.sess.features.borrow().pub_restricted { /* XXX */ }
```

这个`pub_restricted`如果不存在特征标志，则字段（显然以特征命名）通常为假；如果存在，则为真。因此，转换代码以假定字段为真。在这种情况下，这意味着移除`if`然后离开`/* XXX */`.

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
