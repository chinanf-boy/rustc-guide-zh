# 附录D：代码索引

rustc有很多重要的数据结构。这是尝试提供一些指导，以便在何处了解有关编译器的一些关键数据结构的更多信息。

| 项目 | 类 | 简短的介绍 | 章节 | 宣言 |
| --- | --- | ----- | --- | --- |
| `BodyId` | 结构 | 四种类型的HIR节点标识符之一 | [HIR中的标识符] | [SRC / librustc / HIR / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.BodyId.html) |
| `CompileState` | 结构 | 在每个编译器传递时传递给回调的状态 | [Rustc驱动程序] | [SRC / librustc_driver / driver.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/driver/struct.CompileState.html) |
| `ast::Crate` | 结构 | 已解析的包的语法级表示 | [解析器] | [SRC / librustc / HIR / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.Crate.html) |
| `hir::Crate` | 结构 | 一个更加抽象，编译器友好的箱子AST形式 | [Hir] | [SRC / librustc / HIR / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Crate.html) |
| `DefId` | 结构 | 四种类型的HIR节点标识符之一 | [HIR中的标识符] | [SRC / librustc / HIR / def_id.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/def_id/struct.DefId.html) |
| `DiagnosticBuilder` | 结构 | 用于构建编译器诊断的结构，例如错误或lints | [发出诊断] | [SRC / librustc_errors / diagnostic_builder.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_errors/struct.DiagnosticBuilder.html) |
| `DocContext` | 结构 | rustdoc在爬过箱子以收集其文档时使用的状态容器 | [Rustdoc] | [SRC / librustdoc / core.rs](https://github.com/rust-lang/rust/blob/master/src/librustdoc/core.rs) |
| `HirId` | 结构 | 四种类型的HIR节点标识符之一 | [HIR中的标识符] | [SRC / librustc / HIR / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.HirId.html) |
| `NodeId` | 结构 | 四种类型的HIR节点标识符之一。被逐步淘汰 | [HIR中的标识符] | [SRC / libsyntax / ast.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.NodeId.html) |
| `P` | 结构 | 拥有不可变的智能指针。相比之下，`&T`不属于，和`Box<T>`不是一成不变的。 | 没有 | [SRC /语法/ ptr.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ptr/struct.P.html) |
| `ParamEnv` | 结构 | 有关通用参数的信息或`Self`，适用于处理相关或通用项目 | [参数环境] | [SRC / librustc / TY / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.ParamEnv.html) |
| `ParseSess` | 结构 | 此结构包含有关解析会话的信息 | [解析器] | [SRC / libsyntax /解析/ mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/struct.ParseSess.html) |
| `Rib` | 结构 | 表示单个范围的名称 | [名称解析] | [SRC / librustc_resolve / lib.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_resolve/struct.Rib.html) |
| `Session` | 结构 | 与编译会话关联的数据 | [解析器]，[Rustc驱动程序] | [SRC / librustc /会话/ mod.html](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/session/struct.Session.html) |
| `SourceFile` | 结构 | 的一部分`SourceMap`。将AST节点映射到单个源文件的源代码。之前称为FileMap | [解析器] | [SRC / libsyntax_pos / lib.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceFile.html) |
| `SourceMap` | 结构 | 将AST节点映射到其源代码。它由...组成`SourceFile`秒。之前被称为CodeMap | [解析器] | [SRC / libsyntax / source_map.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceMap.html) |
| `Span` | 结构 | 用户源代码中的位置，主要用于错误报告 | [发出诊断] | [SRC / libsyntax_pos / span_encoding.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax_pos/struct.Span.html) |
| `StringReader` | 结构 | 这是解析期间使用的词法分析器。它消耗正在编译的原始源代码中的字符，并生成一系列令牌以供解析器的其余部分使用 | [解析器] | [SRC / libsyntax /解析/词法/ mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/lexer/struct.StringReader.html) |
| `syntax::token_stream::TokenStream` | 结构 | 一个抽象的令牌序列，组织成`TokenTree`小号 | [解析器]，[宏观扩张] | [SRC / libsyntax / tokenstream.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/tokenstream/struct.TokenStream.html) |
| `TraitDef` | 结构 | 此结构包含特征的类型信息定义 | [该`ty`模块] | [SRC / librustc / TY / trait_def.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/trait_def/struct.TraitDef.html) |
| `TraitRef` | 结构 | 特征及其输入类型的组合（例如`P0: Trait<P1...Pn>`） | [特质解决：目标和条款:]，[特质解决：降低动作:] | [SRC / librustc / TY / sty.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TraitRef.html) |
| `Ty<'tcx>` | 结构 | 这是用于类型检查的类型的内部表示 | [类型检查] | [SRC / librustc / TY / mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/type.Ty.html) |
| `TyCtxt<'cx, 'tcx, 'tcx>` | 结构 | “打字环境”。这是编译器中的中心数据结构。它是用于执行各种查询的上下文 | [该`ty`模块] | [SRC / librustc / TY / context.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html) |

[the hir]: ../hir.html

[identifiers in the hir]: ../hir.html#hir-id

[the parser]: ../the-parser.html

[the rustc driver]: ../rustc-driver.html

[type checking]: ../type-checking.html

[the `ty` modules]: ../ty.html

[rustdoc]: ../rustdoc.html

[emitting diagnostics]: ../diag.html

[macro expansion]: ../macro-expansion.html

[name resolution]: ../name-resolution.html

[parameter environment]: ../param_env.html

[trait solving: goals and clauses]: ../traits/goals-and-clauses.html#domain-goals

[trait solving: lowering impls]: ../traits/lowering-rules.html#lowering-impls
