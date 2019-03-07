# 宏观扩张

宏展开发生在分析过程中。`rustc`实际上有两个解析器：普通的Rust解析器和宏解析器。在解析阶段，普通的rust解析器将把宏的内容和它们的调用放在一边。稍后，在名称解析之前，将使用代码的这些部分扩展宏。反过来，当宏分析器需要绑定一个元变量（例如`$my_expr`）分析宏调用的内容时。宏扩展代码位于[`src/libsyntax/ext/tt/`][code_dir].本章旨在解释宏观扩张是如何运作的。

### 例子

有一个例子可以参考是很有帮助的。对于本章的其余部分，每当我们提到“示例”时，*定义*“，我们的意思是：

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

`$mvar`被称为*元变量*.与普通变量不同，元变量不是绑定到计算中的值，而是绑定到*在编译时*到一棵树*令牌*.  一*令牌*是语法的单个“单位”，例如标识符（例如`foo`）或标点符号（例如`=>`）还有其他特殊的令牌，例如`EOF`，表示不再有令牌。由成对括号（如字符）生成的标记树（`(`…`)`，`[`…`]`和`{`…`}`）–它们包括打开和关闭以及介于两者之间的所有标记（我们确实需要像字符这样的括号保持平衡）。宏扩展操作在令牌流上，而不是源文件的原始字节上，这会降低很多复杂性。宏扩展器（以及编译器的其余部分）实际上并不关心代码中某些语法结构的确切行和列；它关心代码中使用的是什么构造。使用令牌可以让我们关心*什么*不用担心*在哪里？*. 有关令牌的更多信息，请参阅[句法分析][parsing]这本书的一章。

每当我们提到“示例*调用*“，我们的意思是：

```rust,ignore
printer!(print foo); // Assume `foo` is a variable defined somewhere else...
```

将宏调用扩展到语法树的过程。`println!("{}", foo)`然后将其扩展为呼叫`Display::fmt`被称为*宏扩展*，这是本章的主题。

### 宏分析器

宏扩展有两个部分：解析定义和解析调用。有趣的是，两者都是由宏解析器完成的。

基本上，宏解析器类似于基于NFA的regex解析器。它使用的算法在精神上类似于[Earley解析算法](https://en.wikipedia.org/wiki/Earley_parser). 宏分析器在中定义[`src/libsyntax/ext/tt/macro_parser.rs`][code_mp].

宏解析器的接口如下（稍微简化了一点）：

```rust,ignore
fn parse(
    sess: ParserSession,
    tts: TokenStream,
    ms: &[TokenTree]
) -> NamedParseResult
```

在此界面中：

-   `sess`是一个“解析会话”，它跟踪一些元数据。最值得注意的是，它用于跟踪生成的错误，以便向用户报告这些错误。
-   `tts`是一个令牌流。宏解析器的工作是使用原始令牌流，并将元变量绑定输出到相应的令牌树。
-   `ms`一*匹配器*. 这是一系列要匹配的令牌树`tts`反对。

在regex解析器的类比中，`tts`是输入，我们正在将其与模式匹配`ms`. 用我们的例子，`tts`可能是包含示例调用内部的令牌流`print foo`，同时`ms`可能是标记序列（树）`print $mvar:ident`.

分析器的输出是`NamedParseResult`，表示发生了以下三种情况中的哪一种：

-   成功：`tts`匹配给定的匹配者`ms`，我们已经生成了一个从元变量到相应标记树的绑定。
-   失败：`tts`不匹配`ms`. 这将导致一条错误消息，例如“没有规则需要令牌”*瞎说*“。
-   错误：发生了一些致命错误*在解析器中*. 例如，如果有多个模式匹配，则会发生这种情况，因为这表示宏不明确。

定义了完整接口[在这里][code_parse_int].

宏解析器与普通的regex解析器几乎完全相同，但有一个例外：为了解析不同类型的元变量，例如`ident`，`block`，`expr`等等，宏解析器有时必须调用正常的rust解析器。

如上所述，宏的定义和调用都是使用宏解析器进行解析的。这是非常不直观和自我参照的。要分析宏的代码*定义*是在[`src/libsyntax/ext/tt/macro_rules.rs`][code_mr]. 它将用于匹配宏定义的模式定义为`$( $lhs:tt => $rhs:tt );+`. 换句话说，a`macro_rules`定义的主体中应至少出现一个标记树，后跟`=>`后面是另一个令牌树。当编译器`macro_rules`定义，它使用此模式来匹配宏定义中每个规则的两个标记树*使用宏分析器本身*. 在我们的示例定义中，元变量`$lhs`将匹配双臂的图案：`(print $mvar:ident)`和`(print twice $mvar:ident)`.  和`$rhs`将与两臂的身体相匹配：`{ println!("{}", $mvar); }`和`{
println!("{}", $mvar); println!("{}", $mvar); }`. 解析器将在需要扩展宏调用时保留这些知识。

当编译器进行宏调用时，它使用上面描述的基于NFA的宏分析器来解析该调用。但是，使用的匹配器是第一个令牌树（`$lhs`）从宏的手臂中提取*定义*. 使用我们的示例，我们将尝试匹配令牌流`print
foo`从对Matchers的调用中`print $mvar:ident`和`print
twice $mvar:ident`我们之前从定义中提取的。算法是完全相同的，但是当宏解析器到达当前匹配器中需要匹配的位置时，*非终结符*（例如）`$mvar:ident`，它调用普通的rust解析器来获取该非终端的内容。在这种情况下，rust解析器将查找`ident`它发现的令牌（`foo`）然后返回宏分析器。然后，宏解析器按正常方式进行解析。另外，请注意，来自不同手臂的matcher中只有一个与调用匹配；如果有多个匹配，则解析不明确；如果没有匹配，则存在语法错误。

有关宏分析器实现的详细信息，请参阅中的注释。[`src/libsyntax/ext/tt/macro_parser.rs`][code_mp].

### 卫生用品

如果你曾经使用C/C++预处理器宏，你知道有一些恼人的和难以调试的GoCHAAS！例如，考虑以下C代码：

```c
#define DEFINE_FOO struct Bar {int x;}; struct Foo {Bar bar;};

// Then, somewhere else
struct Bar {
    ...
};

DEFINE_FOO
```

大多数人都避免这样编写C——这是有充分理由的：它不能编译。这个`struct Bar`由宏定义的名称与`struct
Bar`在代码中定义。还要考虑以下示例：

```c
#define DO_FOO(x) {\
    int y = 0;\
    foo(x, y);\
    }

// Then elsewhere
int y = 22;
DO_FOO(y);
```

你看到问题了吗？我们想打个电话`foo(22, 0)`但是我们得到了`foo(0, 0)`因为宏定义了自己的`y`！

这两个都是*宏观卫生*问题。*卫生用品*与如何处理定义的名称相关*在宏中*.特别是，卫生宏系统可以防止由于宏中引入的名称而导致的错误。生锈宏是卫生的，因为它们不允许人们写上面的各种错误。

在较高的层次上，Rust编译器内部的卫生是通过跟踪引入和使用名称的上下文来实现的。然后我们可以根据上下文消除名称的歧义。宏系统的未来迭代将允许对宏作者使用该上下文进行更大的控制。例如，宏作者可能希望在调用宏的上下文中引入一个新名称。或者，宏作者可能正在定义一个仅在宏内使用的变量（即，它不应在宏外可见）。

在rustc中，这个“上下文”通过`Span`s.

托多：什么叫现场卫生？什么是DEF现场卫生？

托多

### 程序宏

托多

### 自定义派生

托多

托多：也许是关于宏2.0的？

[code_dir]: https://github.com/rust-lang/rust/tree/master/src/libsyntax/ext/tt

[code_mp]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_parser/

[code_mr]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_rules/

[code_parse_int]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ext/tt/macro_parser/fn.parse.html

[parsing]: ./the-parser.html
