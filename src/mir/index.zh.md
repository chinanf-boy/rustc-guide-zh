# MIR（中层红外）

米尔是Rust的*中级中级代表*. 它是由[希尔](../hir.html). MIR是在[RFC 1211]. 这是一种从根本上简化的铁锈形式，用于某些流量敏感的安全检查，尤其是取土检查！–也用于优化和代码生成。

如果您想对mir进行非常高级的介绍，以及它所依赖的一些编译器概念（如控制流图和去散乱），您可能会喜欢[Rust Lang介绍Mir的博客帖子][blog].

[blog]: https://blog.rust-lang.org/2016/04/19/MIR.html

## MIR简介

mir在[`src/librustc/mir/`][mir]模块，但操作它的大部分代码都在[`src/librustc_mir`][mirmanip].

[rfc 1211]: http://rust-lang.github.io/rfcs/1211-mir.html

MIR的一些关键特征是：

-   它是基于[控制流程图][cfg].
-   它没有嵌套表达式。
-   mir中的所有类型都是完全明确的。

[cfg]: ../appendix/background.html#cfg

## 关键MIR词汇

本节介绍MIR的关键概念，总结如下：

-   **基本块**：控制流程图的单位，包括：
    -   **声明：**具有一个继承者的操作
    -   **终结者：**具有潜在多个继承者的操作；总是在块的末尾
    -   （如果你不熟悉这个术语*基本块*见[背景章节][cfg]）
-   **当地人：**在堆栈上分配的内存位置（至少在概念上），如函数参数、局部变量和临时变量。它们由索引标识，用前导下划线编写，如`_1`. 还有一个特别的“本地”（`_0`）分配用于存储返回值。
-   **地点：**在内存中标识位置的表达式，例如`_1`或`_1.f`.
-   **Rvalues：**产生值的表达式。“R”代表的是这些是一项任务的“右侧”。
    -   **操作数：**右值的参数，可以是常量（如`22`）或者一个地方（比如`_1`）

您可以通过将简单程序转换为mir并读取漂亮的打印输出来了解mir的结构。事实上，游乐场让这很容易，因为它提供了一个mir按钮，可以为您的程序显示mir。尝试运行此程序（或[单击此链接][sample-play]，然后单击顶部的“mir”按钮：

[sample-play]: https://play.rust-lang.org/?gist=30074856e62e74e91f06abd19bd72ece&version=stable

```rust
fn main() {
    let mut vec = Vec::new();
    vec.push(1);
    vec.push(2);
}
```

您应该看到如下内容：

```mir
// WARNING: This output format is intended for human consumers only
// and is subject to change without notice. Knock yourself out.
fn main() -> () {
    ...
}
```

这是MIR格式`main`功能。

**变量声明。**如果我们再深入一点，就会看到它以一系列变量声明开始。它们看起来像这样：

```mir
let mut _0: ();                      // return place
scope 1 {
    let mut _1: std::vec::Vec<i32>;  // "vec" in scope 1 at src/main.rs:2:9: 2:16
}
scope 2 {
}
let mut _2: ();
let mut _3: &mut std::vec::Vec<i32>;
let mut _4: ();
let mut _5: &mut std::vec::Vec<i32>;
```

你可以看到mir中的变量没有名字，它们有索引，比如`_0`或`_1`.我们还混合了用户变量（例如，`_1`）具有临时值（例如，`_2`或`_3`）。您可以分辨用户定义变量之间的区别，其中有一个注释可以给出它们的原始名称。（`// "vec" in scope 1...`）。“范围”块（例如，`scope 1 { .. }`）描述源程序的词汇结构（名称在作用域中时）。

**基本块。**进一步阅读，我们看到我们的第一个**基本块**（当然，当你看到它时，它看起来可能有点不同，我忽略了一些评论）：

```mir
bb0: {
    StorageLive(_1);
    _1 = const <std::vec::Vec<T>>::new() -> bb2;
}
```

基本块由一系列**声明**还有决赛**终结者**.在这种情况下，有一种说法：

```mir
StorageLive(_1);
```

此语句指示变量`_1`是“活的”，意思是它可以在以后使用-这将持续到我们遇到`StorageDead(_1)`语句，指示变量`_1`已完成使用。LLVM使用这些“存储语句”来分配堆栈空间。

这个**终结者**街区的`bb0`是打电话给`Vec::new`以下内容：

```mir
_1 = const <std::vec::Vec<T>>::new() -> bb2;
```

终结符不同于语句，因为它们可以有多个继承者——也就是说，控制权可能流向不同的地方。函数调用，如调用`Vec::new`总是因为有可能展开而终止，尽管在`Vec::new`我们可以看到，实际上不可能展开，因此我们只列出一个辅助块，`bb2`.

如果我们期待`bb2`，我们将看到如下情况：

```mir
bb2: {
    StorageLive(_3);
    _3 = &mut _1;
    _2 = const <std::vec::Vec<T>>::push(move _3, const 1i32) -> [return: bb3, unwind: bb4];
}
```

这里有两种说法：另一种`StorageLive`，介绍`_3`临时的，然后是任务：

```mir
_3 = &mut _1;
```

一般来说，作业的形式如下：

```text
<Place> = <Rvalue>
```

一个地方就像一个表达`_3`，请`_3.f`或`*_3`–表示内存中的位置。一个**价值**是一个创建值的表达式：在本例中，rValue是一个可变的borrow表达式，它看起来像`&mut <Place>`.因此，我们可以这样定义rvalues的语法：

```text
<Rvalue>  = & (mut)? <Place>
          | <Operand> + <Operand>
          | <Operand> - <Operand>
          | ...

<Operand> = Constant
          | copy Place
          | move Place
```

从这个语法中可以看到，rvalue不能嵌套——它们只能引用位置和常量。此外，当你使用一个地方时，我们会指出我们是否**复制它**（要求该地方有一个类型`T`在哪里？`T: Copy`）或**移动它**（适用于任何类型的场所）。例如，如果我们有`x
= a + b + c`在Rust中，这将编译成两个语句和一个临时语句：

```mir
TMP1 = a + b
x = TMP1 + c
```

（[试试看][play-abc]尽管您可能希望执行释放模式以跳过溢出检查。）

[play-abc]: https://play.rust-lang.org/?gist=1751196d63b2a71f8208119e59d8a5b6&version=stable

## miR数据类型

MIR数据类型在[`src/librustc/mir/`][mir]模块。上一节中提到的每一个关键概念都以一种非常简单的方式映射到一个锈斑类型。

MIR的主要数据类型是`Mir`. 它包含单个函数的数据（以及“提升常量”的mir子实例），但是[你可以阅读下面的内容](#promoted)）

-   **基本块**：基本块存储在字段中`basic_blocks`；这是`BasicBlockData`结构。没有人直接引用基本块：相反，我们传递`BasicBlock`值，即[新类型']索引到这个向量。
-   **声明**由类型表示`Statement`.
-   **终结者**代表为`Terminator`.
-   **当地人**由一个[新类型']索引类型`Local`. 局部变量的数据位于`Mir` (the `local_decls`向量）。还有一个特殊的常数`RETURN_PLACE`标识表示返回值的特殊“本地”。
-   **地方**由枚举标识`Place`. 有几种变体：
    -   局部变量，如`_1`
    -   静态变量`FOO`
    -   **投影**它们是字段或其他从基础位置“突出”的内容。例如，这个地方`_1.f`是一个投影，`f`是“投影元素”`_1`是基本路径。`*_1`也是一个投影，用`*`代表`ProjectionElem::Deref`元素。
-   **价值观**由枚举表示`Rvalue`.
-   **操作数**由枚举表示`Operand`.

## 表示常量

*被书写*

<a name="promoted"></a>

### 提升常量

*被书写*

[mir]: https://github.com/rust-lang/rust/tree/master/src/librustc/mir

[mirmanip]: https://github.com/rust-lang/rust/tree/master/src/librustc_mir

[mir]: https://github.com/rust-lang/rust/tree/master/src/librustc/mir

[newtype'd]: ../appendix/glossary.html
