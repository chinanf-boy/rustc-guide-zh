# rustdoc 的徒步旅行

rustdoc 实际上直接使用 Rustc 的内部功能。它与编译器和标准库一起存在于 rust 树中。这一章是关于它是如何工作的。

rustdoc 完全在[`librustdoc`][rd]箱子内实现。它运行编译器，直到我们有一个箱子（HIR）的内部描述，并且有运行一些关于某些 items 类型查询的能力。[HIR]和[查询]在相关章节中，有所讨论。

[hir]: ./hir.zh.html
[queries]: ./query.zh.html
[rd]: https://github.com/rust-lang/rust/tree/master/src/librustdoc

在此之后，`librustdoc`执行两个主要步骤来渲染一组文档：

- 将 AST“清理”成更适合将要创建文档的形式（并且稍微更能抵抗编译器中的混乱）。
- 使用这个已清理的 AST 渲染一个箱子的文档，一次一页。

当然，不仅仅是这个，这些描述简化了很多细节，但这是高层级的概述。

（旁注：`librustdoc`是库类型的箱子！这个`rustdoc`二进制文件是使用[`src/tools/rustdoc`][bin]项目创建的。虽然注意到字面上，所做的就是在这个箱子里，调用库类型`lib.rs`的`main()`，就是如此。）

[bin]: https://github.com/rust-lang/rust/tree/master/src/tools/rustdoc

## 备忘单

- 使用`./x.py build --stage 1 src/libstd src/tools/rustdoc`要生成可用的 rustDoc，这样可以在其他项目上运行。
  - 添加`src/libtest`，让`rustdoc --test`能够使用。
  - 如果你之前用过`rustup toolchain link local /path/to/build/$TARGET/stage1`，然后在上一个构建命令之后，用`cargo +local doc`就行了。
- 输入`./x.py doc --stage 1 src/libstd`命令，运用 rustdoc 生成标准库文档。
  - 已完成的文档(捆绑包)将在`build/$TARGET/doc/std`目录，尽管捆绑包的使用方式，就是复制`doc`文件夹到 Web 服务器，因为这就是 CSS/JS 和登录页应该在的位置。
- HTML 大多数的打印代码都在`html/format.rs`和`html/render.rs`。 它一堆的`fmt::Display`实现和补充功能。
- 上面实现了`Display`的类型，定义在`clean/mod.rs`，就在定制`Clean`trait 的旁边，过去常常用来处理掉 Rustc HIR。
- 使用 rustdoc 作为测试线束的特定位(bits)，在`test.rs`.
- Markdown 渲染器已`html/markdown.rs`准备好，包括从给定的 markodwn 块中提取 doctest 的函数。
- rustdoc *输出*的测试，位于`src/test/rustdoc`，由 rustbuild 的测试运行程序和补充脚本`src/etc/htmldocck.py`控制。
- 搜索生成索引的测试位于`src/test/rustdoc-js`，作为对标准库搜索索引和预期结果的查询进行编码的一系列 javascript 文件。

## 从箱子到 Clean

在`core.rs`有两个中心：`DocContext`结构，和`run_core`函数。后者是 rustdoc 请求 Rustc ，编译一个箱子到 rustdoc 可以接管的位置。前者是一种状态容器，在'爬'箱子和收集它文档时，会用到。

爬箱子的主要过程在`clean/mod.rs`完成，具体通过其中定义的几个`Clean`trait 实现。下面是一个转换 trait，它定义了一种方法：

```rust,ignore
pub trait Clean<T> {
    fn clean(&self, cx: &DocContext) -> T;
}
```

`clean/mod.rs`还定义了稍后用于渲染文档页面的“cleaned”AST 的类型。每个类型通常配一个`Clean`实现，从 rustc 中提取一些 AST 或 HIR 类型，并将其转换为适当的“cleaned”类型。模块或关联项等大型“项可能在自个的`Clean`实现中，有些额外处理，但在大多数情况下，这些实现是直接转换的。此模块的“入口处”是`impl Clean<Crate> for visit_ast::RustdocVisitor`，由上面的`run_core`调用。

其实嘛，我其实早些时候撒了谎：在`clean/mod.rs`的事件发生之前，有其他的 AST 转换会先发生。举个例子，在`visit_ast.rs`是`RustdocVisitor`结构类型，关乎爬一个`hir::Crate`，去获取第一个 IR(中间描述)的*实际*操作，IR 在`doctree.rs`有所定义。 此过程主要是为了围绕 HIR 类型获得一些中间包装，并处理可见性和内联性。这里就是处理`#[doc(inline)]`，`#[doc(no_inline)]`和`#[doc(hidden)]`的地方，以及有关`pub use`的逻辑，其是否应该得到整页，或模块页面的一个“Reexport”行。

另一件发生在`clean/mod.rs`的要事，是把文档注释组合和`#[doc=""]`属性搞进 Attributes 结构的单独字段中，如此一来，就能提供任何手写文档。这让稍后的过程中，此文档的收集更加容易。

这个过程的主要输出是，一个`clean::Crate`加个 items 树，用来描述目标箱子中，公有-文档化的 items。

### 烫手山芋

在进入下一个主要步骤之前，文档中会出现一些重要的“传递参数”。这些参数，可以将单独的“属性”组合成一个字符串，并去掉前空格，以使文档在 markdown 分析器上更容易使用，或者删除不公开或故意隐藏的`#[doc(hidden)]`项。 这些都是在`passes/`目录中实现的，一个参数一个文件。默认情况下，所有这些参数都在一个箱子上运行，但是可以通过传递`--document-private-items`给 rustdoc ，让私有/隐藏的项通行。注意，与前面的一组 AST 转换不同，传递参数发生在*cleaned*箱子中。

（严格来说，你可以微调参数的执行，甚至添加自己的参数，但是[我们正试图反对这一点。][44136]。 如果您需要对这些参数，进行更精细的粒度控制，请通知我们！）

[44136]: https://github.com/rust-lang/rust/issues/44136

以下是当前（截至本文撰写时）参数列表：

- `propagate-doc-cfg`——传播`#[doc(cfg(...))]`到子项。
- `collapse-docs`将所有文档属性联到一个文档属性中。因为每行文档注释都作为单独的文档属性提供，所以将它们组合成一个以换行符间隔的字符串，是有必要的。
- `unindent-comments`删除注释上多余的缩进，以便 markdown 好喜欢它。这是必要的，因为编写文档的惯例是在`///`(或`//!`)标记和文本之间加个空格，方便人类阅读。而去掉前空格，使文本更容易被 markdown 分析器解析。（在过去，所使用的 markdown 解析器不符合通用标记，对额外的空格，感到烦扰，但在今时今日，似乎不太成问题。）
- `strip-priv-imports`从一个箱子中，删除所有私有导入语句（`use`，`extern crate`）。这是必要的，因为 rustdoc 会处理*公有的*导入，通过：将项目文档导入模块，或创建包含导入内容的“Reexports”部分来导入。该参数确保所有这些导入，真真切切都与文档相关联。
- `strip-hidden`和`strip-private`：剥离所有的`doc(hidden)`以及输出的私有项。`strip-private`暗指`strip-priv-imports`。 基本上，目标是删除与公有文档无关的项。

## 从 clean 到箱子

这就是 rustdoc 的“第二阶段”开始的地方。这个阶段主要生活在`html/`文件夹，一切都在`html/render.rs`里面的`run()`开始。 此代码负责设置`Context`，`SharedContext`和`Cache`，这些会在渲染过程中使用；复制每个渲染文档集合的静态文件（如 fonts、css 和 javascript，都在`html/static/`存放）；创建搜索索引，并在开始渲染渲染箱子的所有文档之前，打印出源代码渲染情况。

直接在`Context`上实现几个函数，拿上`clean::Crate`，并在渲染项或递归模块的子项之间，设置一些状态。“页面渲染”从这里开始，通过在`html/layout.rs`调用一个巨大的`write!()`。 项目和文档实际生成 HTML 的这部分过程，发生在一系列`std::fmt::Display`实现和函数中，其函数传递一个`&mut std::fmt::Formatter`参数(注意`format!`)。 绘出页面主体(body)的顶层实现是`impl<'a> fmt::Display for Item<'a>`，在`html/render.rs`里面，它会在多个基于要渲染的`Item`类型的`item_*`函数，选一个来切换。

根据您要渲染的代码类型，可能会是使用`html/render.rs`，其针对诸如“结构页应该打印哪些部分”之类的大项，或针对较小组件的`html/format.rs`，如“我应该如何将一个 WHERE 子句，作为其他项的一部分打印出来”。

每当 rustdoc 遇到需要打印手写文档的项目时，它就会调用`html/markdown.rs`，其是与 markdown 解析器的接口。它公开为一系列类型，这些类型包装了一个 markdown 字符串，并实现了`fmt::Display`，能够发出 HTML 文本。要特别注意启用某些功能，如脚注和表格，并在运行 markdown 分析器之前，向 rust 代码块添加语法高亮（通过`html/highlight.rs`）。这里还有一个函数（`find_testable_code`）它专门扫描 RUst 代码块，这样，测试运行程序(test-runner)就可以在箱子中找到所有的文档测试。

### 从汤到坚果

（替代标题：[“有一条不可破的线，把我们和第一个'细胞'联系在一起][video]）

[video]: https://www.youtube.com/watch?v=hOLAGYmUQV0

需要注意的是，AST clean 时可以向编译器询问信息（关键，`DocContext`包含一个`TyCtxt`)，但页面渲染时不能这样做。这个`clean::Crate`在`run_core`内创建，在交给`html::render::run`之前，传到了编译器上下文之外。这意味着许多“补充数据”，在一个项的定义中，不能立即得到，比如哪个 trait 被语言用做`Deref`trait，这都需要在 clean 过程中收集，存储在`DocContext`，然后在 HTML 渲染期间，传给`SharedContext`。这样，需要的'食材'为一组共享状态、上下文变量和`RefCell`s.

另外需要注意的是，一些来自“询问编译器”，而得到的项，不会直接进入`DocContext`—— 例如，当从外部箱子装载项时，rustdoc 将询问 trait 实现，并基于该信息，生成代表实现代码块的新`Item`们。这是直接进入返回的`Crate`，而不是通过`DocContext`迂回进入。这样，这些实现就可以在渲染 HTML 之前，与其他实现一起被收集。

## 其他花招

所有这些都描述了从 Rust 箱子生成 HTML 文档的过程，但 rustdoc 还使用了其他两种主要模式。一是可以在独立的 markdown 文件上运行，二也可以在 Rust 代码或独立 markdown 文件上运行 doctest。对于前者，它直接通向`html/markdown.rs`，还可选择将目录插入到 HTML 输出。

拿后者来说，rustdoc 运行类似的部分编译，来获取`test.rs`的有关文档，但是，它不需要经过完整的 clean 和渲染过程，而是运行一个更简单的箱子，走一遍，*仅*抓取手写文件。与上述在`html/markdown.rs`提到的`find_testable_code`相结合，在把测试交给 libtest 运行程序之前，它构建了一组要运行的测试。`make_test`函数是`test.rs`文件的重点，就是手写文档测试，转化成可执行的地方。

`make_test`的一些额外阅读，可以[在这里](https://quietmisdreavus.net/code/2018/02/23/how-the-doctests-get-made/)找到。

## 一丝不苟，注重细节

简而言之，这就是 rustdoc 的代码，但存储库中，还有很多要处理的事情。既然我们手中，已经有了`compiletest`套件，里面`src/test/rustdoc`有一套测试，确保最终的 HTML 能在各种情况下，成为我们所期望的那样。这些测试还使用了补充脚本，`src/etc/htmldocck.py`，这样就可以使用 XPath 符号查看最终的 HTML，从而精确地查看输出。rustdoc 测试中所有可用的命令，在`htmldocck.py`有完整描述。

此外，还对搜索索引和 rustdoc 查询索引的能力，进行了单独的测试。`src/test/rustdoc-js`中的每个文件，都包含一个不同的搜索查询和预期结果，按“搜索”选项卡进行细分。这些文件由`src/tools/rustdoc-js`的一个脚本和 node.js 运行时，进行处理。这些测试没有写得那么透彻，但是在`basic.js`，有个概括示例：找到所有标签中的功能结果。基本的想法是你匹配一个给定的`QUERY`，再配一套`EXPECTED`结果，包括每个目的完整项路径。
