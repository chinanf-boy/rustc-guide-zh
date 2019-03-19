# 宏扩展

宏扩展发生在解析过程中。`rustc`实际上有两个解析器：普通的 Rust 解析器和宏解析器。在解析阶段，普通的 rust 解析器将把宏的内容和它们的调用放在一边。稍后，在名称解析之前，将使用这些代码的一部分，来扩展宏。反过来，当宏解析器需要绑定一个元变量（例如`$my_expr`），用来解析宏调用的内容时，可能你会调用普通的 Rust 解析器。宏扩展代码位于[`src/libsyntax/ext/tt/`][code_dir]。本章旨在，解释宏扩展是如何运作的。

### 例子

来，有个参考例子，有所帮助些。对于本章的其余部分，每当我们提到“_定义(definition)_ 示例”时，我们的意思是：

```rust,ignore
macro_rules! printer {
    (print $mvar:ident) => {
        println!("{}", $mvar);
    }
    (print twice $mvar:ident) => {
        println!("{}", $mvar);
        println!("{}", $mvar);
    }
}
```

`$mvar`被称为*元变量(metavariable)*。与普通变量不同，元变量不是绑定到计算中的值，而是*在编译时*，绑定到树中的一个*token*。 一个*token*是语法的一个“单位”，例如一个标识符（例如`foo`）或标点符号（例如`=>`）还有其他特殊的 token，例如`EOF`表示：这里后面不再有 token 了。由成对括号（如字符（`(`…`)`，`[`…`]`和`{`…`}`））生成的 token 树 —— 会包括开符和关符，以及介于两者之间的所有 token（我们确实需要让这样括号的字符，凑对，保持平衡）。宏扩展不是源文件的原始字节上，而是在 token 流上操作，这会降低很多复杂性。宏扩展器（以及编译器的其余部分）实际上并不关心代码中，某些语法结构的确切行和列；它只关心代码中，使用什么构造。使用 token 可以让我们关心*什么*，而不用担心*在哪里*。 有关 token 的更多信息，请参阅这本书的[解析][parsing]一章。

每当我们提到“*调用(invocation)*示例“，我们的意思是：

```rust,ignore
printer!(print foo); // 假设`foo`在某处定义为一个变量
```

把宏调用扩展到语法树`println!("{}", foo)`，然后将其扩展进一个`Display::fmt`调用的过程，被称为*宏扩展*，和而这正是本章的主题。

### 宏解析器

宏扩展有两个部分：解析所定义的，和解析所调用的。有趣的是，两者都是由宏解析器完成的。

基本上，宏解析器类似，基于 NFA 的 regex 解析器。它使用的算法在思路上，类似[Earley 解析算法](https://en.wikipedia.org/wiki/Earley_parser)。 宏解析器在[`src/libsyntax/ext/tt/macro_parser.rs`][code_mp]中定义。

宏解析器的接口如下（稍微简化了一点）：

```rust,ignore
fn parse(
    sess: ParserSession,
    tts: TokenStream,
    ms: &[TokenTree]
) -> NamedParseResult
```

在此接口中：

- `sess`是一个“解析会话(session)”，它跟踪一些元数据。最值得注意的是，它用于跟踪生成的错误，方便向用户报告这些错误。
- `tts`是一个 token 流。宏解析器的工作是使用原始 token 流，将输出，从元变量到相应的 token 树的绑定。
- `ms`是一个*匹配器(matcher)*。 这是要与`tts`对应匹配的，一系列 token 树。

类比 regex 解析器，`tts`是输入，我们正在用模式`ms`与之匹配。 用我们的例子，`tts`可能是包含，**调用示例内部 `print foo`**的 token 流，同时`ms`可能是一串 `print $mvar:ident` token （树），所以就是`print $mvar:ident`匹配上了`print foo`。

该解析器的输出是`NamedParseResult`，表示发生了以下三种情况的一种：

- 成功(Success)：`tts`匹配到了给予的`ms`，我们已经生成了一个从元变量到相应 token 树的绑定。
- 失败(Failure)：`tts`不匹配`ms`。 这将导致一条错误消息，例如“没有规则需要 token blah（瞎话）“。
- 错误(Error)：*在解析器中*发生了一些致命错误。 例如，如果有多个模式匹配，则会发生这种情况，因为这宏表示不明确。

完整接口定义[在这里][code_parse_int]。

宏解析器与普通的 regex 解析器几乎完全相同，但有一个例外：为了解析不同类型的元变量，例如`ident`，`block`，`expr`等等，宏解析器有时，必须调用普通的 Rust 解析器。

如上所述，宏的定义(definitions)和调用(invocations)，都是使用宏解析器进行解析的。这是非常不直观，和没有对照组的。要解析宏*定义*的代码是在[`src/libsyntax/ext/tt/macro_rules.rs`][code_mr]：它定义了模式匹配，用来匹配一个宏定义，如`$( $lhs:tt => $rhs:tt );+`这样的。 换句话说，一个`macro_rules`定义的主体中，应至少出现一个 token 树，后跟`=>`，再后是另一个 token 树。当编译器运行到`macro_rules`定义，它使用此模式来匹配宏定义中，两个 token 树的每个*使用宏解析器本身*的规则。 在我们的*定义示例*中，元变量`$lhs`将匹配两个语句模式：`(print $mvar:ident)`和`(print twice $mvar:ident)`。 还有`$rhs`将与两个语句模式的主体相匹配：`{ println!("{}", $mvar); }`和`{ println!("{}", $mvar); println!("{}", $mvar); }`。 解析器需要扩展宏调用时，会保留这些'知识'，

当编译器进行宏调用时，它使用上面描述的，基于 NFA 的宏解析器来解析该调用。但是，使用的 matcher 是，从宏*定义*的匹配模式中，提取的第一个 token 树（`$lhs`）。 回到我们的示例，我们将尝试用，调用阶段的 token 流`print foo`，匹配之前宏定义的匹配器`print $mvar:ident`和`print twice $mvar:ident`。算法是完全相同的，但是当宏解析器，到达当前匹配器中，需要匹配 _非终结符_（例如`$mvar:ident`) 的位置时候，它调用普通的 rust 解析器来获取该非终结的内容。在这种情况下，rust 解析器会查找一个`ident`token，这时它找到了（`foo`），然后返回宏解析器。然后，宏解析器按正常方式进行解析。另外，请注意，来自 matcher 中不同匹配模式，只有一个与调用(阶段)匹配；如果有多个匹配，则解析不明确；如果没有匹配，则存在语法错误。

有关宏解析器实现的详细信息，请参阅[`src/libsyntax/ext/tt/macro_parser.rs`][code_mp]中的注释。

### Hygiene(卫生/+安全)

如果你曾经使用 C/C++预处理器的宏，你知道有一些恼人的，和难以调试的 gotchas(陷阱)！例如，考虑以下 C 代码：

```c
#define DEFINE_FOO struct Bar {int x;}; struct Foo {Bar bar;};

// 然后 其他地方
struct Bar {
    ...
};

DEFINE_FOO
```

大多数人都避免这样编写 C —— 这是有充分理由的：它不能编译。这个由宏定义`struct Bar`名称，它与在代码中`struct Bar`定义撞车了。还要考虑以下示例：

```c
#define DO_FOO(x) {\
    int y = 0;\
    foo(x, y);\
    }

// 其他
int y = 22;
DO_FOO(y);
```

你看到问题了吗？我们想调用一个`foo(22, 0)`，但是我们得到了`foo(0, 0)`，这是因为宏定义了自己的`y`！

这两个都是*宏的卫生安全*问题。_在宏中_，*Hygiene*与如何处理定义的名称相关。特别是，卫生的宏系统可以防止由于宏中引入的名称，而导致的错误。Rust 宏是卫生的，因为它们不允许，人们写上面的各种错误。

在较高的层次上，Rust 编译器内部的卫生，是通过跟踪引入，和使用名称的上下文来实现的。然后我们可以根据上下文消除名称的歧义。宏系统的未来迭代，将允许对宏作者使用该上下文进行更大的控制。例如，宏作者可能希望在调用宏的上下文中引入一个新名称。或者，宏作者可能正在定义一个,仅在宏内使用的变量（即，它不应在宏外可见）。

在 rustc 中，这个“上下文(context)”通过`Span`们，跟踪。

TODO：什么叫 call-site hygiene？什么是 def-site hygiene？

TODO

### 程序宏

TODO

### 自定义派生

TODO

TODO：也许是关于宏 2.0 的？

[code_dir]: https://github.com/rust-lang/rust/tree/master/src/libsyntax/ext/tt
[code_mp]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_parser/
[code_mr]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_rules/
[code_parse_int]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_parser/fn.parse.html
[parsing]: ./the-parser.html
