# 鲁斯多克徒步旅行

RustDoc实际上直接使用RustC内部。它与编译器和标准库一起存在于树中。这一章是关于它是如何工作的。

RustDoc完全在板条箱内实施。[`librustdoc`][rd].它运行编译器直到我们有一个板条箱（hir）的内部表示，并且能够运行一些关于项目类型的查询。[希尔]和[查询]在相关章节中讨论。

[hir]: ./hir.html

[queries]: ./query.html

[rd]: https://github.com/rust-lang/rust/tree/master/src/librustdoc

`librustdoc`在此之后执行两个主要步骤来呈现一组文档：

-   将AST“清理”成更适合创建文档的形式（并且稍微更能抵抗编译器中的混乱）。
-   使用这个清洁的AST渲染板条箱的文档，一次一页。

当然，不仅仅是这个，这些描述简化了很多细节，但这是高层的概述。

（旁注：`librustdoc`是图书馆的板条箱！这个`rustdoc`二进制文件是使用中的项目创建的。[`src/tools/rustdoc`][bin].请注意，字面上所有做的就是调用`main()`在这个箱子里`lib.rs`尽管如此。）

[bin]: https://github.com/rust-lang/rust/tree/master/src/tools/rustdoc

## 备忘单

-   使用`./x.py build --stage 1 src/libstd src/tools/rustdoc`要生成可用的RustDoc，可以在其他项目上运行。
    -   添加`src/libtest`能够使用`rustdoc --test`.
    -   如果你用过`rustup toolchain link local /path/to/build/$TARGET/stage1`以前，然后在上一个构建命令之后，`cargo +local doc`就行了。
-   使用`./x.py doc --stage 1 src/libstd`使用此RustDoc生成标准库文档。
    -   已完成的文档将在`build/$TARGET/doc/std`尽管捆绑包的使用方式与复制`doc`文件夹到Web服务器，因为这是CSS/JS和登录页所在的位置。
-   大多数HTML打印代码都在`html/format.rs`和`html/render.rs`. 它在一堆`fmt::Display`实施和补充功能。
-   得到的类型`Display`上面的impl定义在`clean/mod.rs`在海关旁边`Clean`用来处理锈迹的性状。
-   使用RustDoc作为测试线束的特定位位于`test.rs`.
-   标记渲染器加载到`html/markdown.rs`包括从给定的标记块中提取doctest的函数。
-   RustDoc的测试*输出*位于`src/test/rustdoc`，由rustbuild的测试运行程序和补充脚本处理`src/etc/htmldocck.py`.
-   搜索索引生成测试位于`src/test/rustdoc-js`，作为对标准库搜索索引和预期结果的查询进行编码的一系列javascript文件。

## 从板条箱到清洁

在`core.rs`有两个中心项目：`DocContext`结构，和`run_core`功能。后者是RustDoc要求RustC编译一个板条箱到RustDoc可以接管的地方。前者是一种状态容器，用于爬过板条箱收集文件。

板条箱爬行的主要过程是`clean/mod.rs`通过`Clean`内定义的特征。这是一个转换特征，它定义了一种方法：

```rust,ignore
pub trait Clean<T> {
    fn clean(&self, cx: &DocContext) -> T;
}
```

`clean/mod.rs`还定义了稍后用于呈现文档页面的“已清理”AST的类型。每个通常伴随着`Clean`它从rustc中提取一些ast或hir类型，并将其转换为适当的“cleaned”类型。模块或关联项等大型“项可能在其`Clean`实现，但在大多数情况下，这些impl是直接转换的。此模块的“入口点”是`impl Clean<Crate> for visit_ast::RustdocVisitor`，由调用`run_core`上面。

你看，我其实早些时候撒了谎：在`clean/mod.rs`.在`visit_ast.rs`是类型`RustdocVisitor`，其中*事实上*爬行`hir::Crate`要获取第一个中间表示，请在`doctree.rs`. 此过程主要是为了围绕HIR类型获得一些中间包装器，并处理可见性和内联。这里就是`#[doc(inline)]`，`#[doc(no_inline)]`和`#[doc(hidden)]`处理，以及`pub use`应该在模块页面中得到整页或“重新导出”行。

另一件主要的事情发生在`clean/mod.rs`是文档注释和`#[doc=""]`属性到attributes结构的单独字段中，出现在任何获得手写文档的内容上。这使得稍后在流程中收集此文档更加容易。

这个过程的主要输出是`clean::Crate`有一个描述目标板条箱中可公开记录项目的项目树。

### 烫手山芋

在进入下一个主要步骤之前，文档中会出现一些重要的“传递”。这些方法可以将单独的“属性”组合成一个字符串，并去掉前导空格，以使文档在降价分析器上更容易使用，或者删除不公开或故意隐藏的项`#[doc(hidden)]`. 这些都是在`passes/`目录，每次传递一个文件。默认情况下，所有这些通行证都在板条箱上运行，但是有关丢弃私人/隐藏物品的通行证可以通过通行证绕过。`--document-private-items`给鲁斯多克。注意，与前面的一组AST转换不同，传递发生在*清洁*机箱。

（严格来说，你可以微调传球，甚至添加自己的传球，但是[我们正试图贬低这一点。][44136]. 如果您需要对这些焊道进行更精细的粒度控制，请通知我们！）

[44136]: https://github.com/rust-lang/rust/issues/44136

以下是当前（截至本文撰写时）通行证列表：

-   `propagate-doc-cfg`-传播`#[doc(cfg(...))]`儿童项目。
-   `collapse-docs`将所有文档属性连接到一个文档属性中。这是必要的，因为文档注释的每一行都作为单独的文档属性提供，并且这将把它们组合成一个字符串，每个属性之间都有换行符。
-   `unindent-comments`删除注释上多余的缩进，以便降价使其更受欢迎。这是必要的，因为编写文档的惯例是在`///`或`//!`标记和文本，去掉前导空格将使文本更容易被标记分析器解析。（在过去，所使用的降价解析器不符合通用标记，这导致了对额外空白的烦扰，但这在今天似乎不太成问题。）
-   `strip-priv-imports`删除所有私有导入语句（`use`，`extern
    crate`）从一个板条箱。这是必要的，因为RustDoc会处理*公众的*通过将项目的文档导入模块或创建包含导入内容的“重新导出”部分来导入。通行证确保所有这些导入实际上都与文档相关。
-   `strip-hidden`和`strip-private`全部剥离`doc(hidden)`以及输出的私有项。`strip-private`暗示`strip-priv-imports`. 基本上，目标是删除与公共文档无关的项目。

## 从干净到板条箱

这就是RustDoc的“第二阶段”开始的地方。这个阶段主要生活在`html/`文件夹，一切都从`run()`在里面`html/render.rs`. 此代码负责设置`Context`，`SharedContext`和`Cache`在渲染过程中使用，复制每个渲染的文档集中的静态文件（如字体、css和javascript`html/static/`）创建搜索索引并打印出源代码呈现，然后开始呈现板条箱的所有文档。

直接在`Context`拿`clean::Crate`并在呈现项或递归模块的子项之间设置一些状态。“页面呈现”从这里开始，通过一个巨大的`write!()`拜访`html/layout.rs`. 从项目和文档中实际生成HTML的部分出现在一系列`std::fmt::Display`传递一个`&mut std::fmt::Formatter`. 写出页面正文的顶级实现是`impl<'a> fmt::Display for
Item<'a>`在里面`html/render.rs`，切换到多个`item_*`基于类型的函数`Item`被渲染。

根据您要查找的呈现代码类型，您可能会在`html/render.rs`对于诸如“结构页应该打印哪些部分”之类的主要项目，或`html/format.rs`对于较小的组件，比如“我应该如何将WHERE子句作为其他项的一部分打印出来”。

每当RustDoc遇到需要打印手写文档的项目时，它就会调用`html/markdown.rs`与降价解析器的接口。它公开为一系列类型，这些类型包装了一个字符串markdown，并实现`fmt::Display`发出HTML文本。要特别注意启用某些功能，如脚注和表格，并向rust代码块添加语法突出显示（通过`html/highlight.rs`）在运行降价分析器之前。这里还有一个函数（`find_testable_code`）它专门扫描铁锈代码块，这样测试运行程序代码就可以在板条箱中找到所有的教义。

### 从汤到坚果

（替代标题：[“一条从第一条线延伸出来的完整的线`Cell`对我们来说][video]）

[video]: https://www.youtube.com/watch?v=hOLAGYmUQV0

需要注意的是，AST清理可以向编译器询问信息（至关重要的是，`DocContext`包含一个`TyCtxt`，但不能进行页面呈现。这个`clean::Crate`内建`run_core`在传递到编译器上下文之外之前`html::render::run`. 这意味着许多“补充数据”在一个项目的定义中不能立即得到，比如哪个特征是`Deref`语言使用的特征，需要在清洗过程中收集，存储在`DocContext`然后传给`SharedContext`在HTML呈现期间。这表现为一组共享状态、上下文变量和`RefCell`s.

另外需要注意的是，一些来自“询问编译器”的项不会直接进入`DocContext`-例如，当从国外板条箱装载项目时，RustDoc将询问特性实现并生成新的`Item`S代表基于该信息的IMPLS。这直接进入返回的`Crate`而不是迂回通过`DocContext`. 这样，这些实现就可以在呈现HTML之前与其他实现一起收集。

## 其他花招

所有这些都描述了从Rust板条箱生成HTML文档的过程，但RustDoc还使用了其他两种主要模式。它也可以在独立的降价文件上运行，也可以在生锈代码或独立降价文件上运行doctest。对于前者，它直接通向`html/markdown.rs`（可选）包括将目录插入到输出HTML的模式。

对于后者，RustDoc运行类似的部分编译来获取`test.rs`但是，它不需要经过完整的清洁和渲染过程，而是运行一个更简单的板条箱行走来抓取*只是*手写文件。与上述“相结合”`find_testable_code`“在`html/markdown.rs`在将测试交给libtest运行程序之前，它构建了一组要运行的测试。一个著名的地点在`test.rs`是函数`make_test`这就是手写的教义转化成可以执行的东西的地方。

一些额外的阅读`make_test`可以找到[在这里](https://quietmisdreavus.net/code/2018/02/23/how-the-doctests-get-made/).

## 打点我的和穿过T的

简而言之，这就是RustDoc的代码，但回购协议中还有更多的事情要处理。既然我们已经吃饱了`compiletest`套房在手，里面有一套测试`src/test/rustdoc`确保最终的HTML是我们在各种情况下所期望的。这些测试还使用了补充脚本，`src/etc/htmldocck.py`这样，它就可以使用xpath符号查看最终的HTML，从而精确地查看输出。RustDoc测试可用的所有命令的完整描述位于`htmldocck.py`.

此外，还对搜索索引和RustDoc查询索引的能力进行了单独的测试。中的文件`src/test/rustdoc-js`每个包含一个不同的搜索查询和预期结果，按“搜索”选项卡进行细分。这些文件由中的脚本处理`src/tools/rustdoc-js`以及node.js运行时。这些测试没有写得那么透彻，但是可以在`basic.js`.基本的想法是你匹配一个给定的`QUERY`用一套`EXPECTED`结果，包括每个项目的完整项目路径。
