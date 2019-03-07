# rust-lang/rustc-guide [![explain]][source] [![translate-svg]][translate-list] 
    
<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name
    


「 desc 」

[中文](./readme.md) | [english](https://github.com/rust-lang/rustc-guide) 


---

## 校对 🀄️

<!-- doc-templite START generated -->
<!-- repo = 'rust-lang/rustc-guide' -->
<!-- commit = '0456aaa9e197e6d3f8349bca6299becb836e4070' -->
<!-- time = '2019-03-01' -->
翻译的原文 | 与日期 | 最新更新 | 更多
---|---|---|---
[commit] | ⏰ 2019-03-01 | ![last] | [中文翻译][translate-list]

[last]: https://img.shields.io/github/last-commit/rust-lang/rustc-guide.svg
[commit]: https://github.com/rust-lang/rustc-guide/tree/0456aaa9e197e6d3f8349bca6299becb836e4070

<!-- doc-templite END generated -->


### 贡献

欢迎 👏 勘误/校对/更新贡献 😊 [具体贡献请看](https://github.com/chinanf-boy/chinese-translate-list#贡献)
        

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

### 目录

<!-- START doctoc -->
<!-- END doctoc -->

这是一项协作工作，旨在构建一个解释rustc如何工作的指南。本指南的目的是帮助新贡献者找到rustc，以及帮助更有经验的人找出他们之前没有使用过的编译器的一些新部分。

[You can read the latest version of the guide here.](https://rust-lang-nursery.github.io/rustc-guide/)

你也可以找到rustdocs[对于编译器本身][rustdocs]有用。

[rustdocs]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/

### 为指南做贡献

该指南今天很有用，但还有很多工作要做。

如果您想帮助改进指南，我们很乐意拥有您！你可以找到很多问题[issue
tracker](https://github.com/rust-lang/rustc-guide/issues)。只需发表您想要解决的问题的评论，以确保我们不会意外地重复工作。如果您认为缺少某些内容，请打开相关问题！

**一般来说，如果你不知道编译器是如何工作的，那不是问题！**在这种情况下，我们要做的是安排一些时间与你交谈**不**知道代码，或谁想与你配对并弄清楚。然后你可以写出你学到的东西。

一般来说，在编写编译器代码的特定部分时，我们建议您链接到编译器的相关部分[rustc rustdocs][rustdocs]。

为了防止意外引入断开的链接，我们使用了`mdbook-linkcheck`。如果安装在您的机器上`mdbook`将自动调用此链接检查器，否则它将发出警告说无法找到它。

```bash
> cargo install mdbook-linkcheck
```

你会需要`mdbook`版`>= 0.2`。`linkcheck`您将在运行时自动运行`mdbook build`。
