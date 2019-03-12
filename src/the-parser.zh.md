# 语法分析器

解析器负责将原始的Rust源代码转换成一种结构化的形式，这种形式便于编译器使用，通常称为[*抽象语法树*][ast]. AST使用`Span`将特定的AST节点链接回其源文本。

大部分解析器位于[LB语法]机箱。

与大多数解析器一样，解析过程由两个主要步骤组成，

-   词汇分析-将字符流转换为符号树流
-   解析-将令牌树转换为AST

这个`syntax`箱子里有几个主要的玩家，

-   一[`SourceMap`]用于将ast节点映射到其源代码
-   这个[AST模块]包含对应于每个AST节点的类型
-   一[`StringReader`]用于将源代码词法转换为令牌
-   这个[分析器模块]和[`Parser`]struct负责将令牌实际解析为ast节点，
-   还有一个[访问模块]用于浏览AST和检查或改变AST节点。

解析器的主要入口点是通过`parse_*`中的函数[分析器模块].他们让你做的事情像[`SourceFile`][sourcefile]（例如，单个文件中的源）进入令牌流，从令牌流创建一个解析器，然后执行解析器以获取`Crate`（根AST节点）。

为了尽可能减少所做的复制量，两个`StringReader`和`Parser`有将他们与父母捆绑在一起的生命周期`ParseSess`.它包含解析时所需的所有信息，以及`SourceMap`本身。

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
