# 语法解析器

解析器负责将原始的 Rust 源代码转换成一种结构化的形式，这种形式便于编译器使用，通常称为[_抽象语法树_][ast]。AST 是内存中 Rust 程序结构的镜像，使用一个`Span`将特定的 AST 节点链接回其源代码文本。

大部分解析器位于[libsyntax]箱。

与大多数解析器一样，解析过程由两个主要步骤组成，

- 词汇(lexical)分析-将字符流转换为 token 树流
- 解析-将 token 树转换为 AST

这个`syntax`箱子里有几个主要的玩家，

- 一[`SourceMap`]，用于将 AST 节点映射到其源代码
- 这个[ast 模块][ast module]，包含每个 AST 节点的对应类型
- 一[`StringReader`]，用于将源代码词法转换为 token
- 这个[解析器模块][parse module]和[`Parser`]结构，负责将 token 实际解析为 ast 节点，
- 还有一个[访问模块][visit module]用于浏览 AST 和检查或改变 AST 节点。

解析器的主要入口点是通过[解析器模块][parser module]中的，多个`parse_*`函数。他们让你能完成，像把一个[`SourceFile`][sourcefile]（例如，一个文件的源代码）转变为 token 流，再从 token 流创建一个解析器，然后执行该解析器以获取`Crate`（AST 的根 节点）。

为了尽可能减少所做的复制量，`StringReader`和`Parser`两个都有，与其父`ParseSess`的生命周期绑在一起。它包含解析时，所需的所有信息，也包括`SourceMap`本身。

[libsyntax]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/index.html
[rustc_errors]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_errors/index.html
[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[`sourcemap`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceMap.html
[ast module]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/index.html
[parser module]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/index.html
[`parser`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/parser/struct.Parser.html
[`stringreader`]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/lexer/struct.StringReader.html
[visit module]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/visit/index.html
[sourcefile]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/source_map/struct.SourceFile.html
