# MIR施工

降低[希尔]到[密尔]对以下（可能不完整）项列表发生：

-   功能和闭合体
-   Initializers`static`和`const`项目
-   枚举判别式的初始值设定项
-   各种胶水和垫片
    -   元组结构初始值设定项函数
    -   下拉代码`Drop::drop`不直接调用函数）
    -   删除没有显式`Drop`实施

通过调用[`mir_built`]查询。中间代表介于[希尔]和[密尔]称为[头发]仅在下降过程中使用。这个[头发]其最重要的特点是，各种调整（在没有显式语法的情况下发生）如强制、autoderef、autoref和重载方法调用都已成为显式强制转换、deref操作、引用表达式或具体函数调用。

这个[头发]具有镜像[希尔]数据类型，但不是`-x`作为一个`hair::ExprKind::Neg(hair::Expr)`这是一个`hair::ExprKind::Neg(hir::Expr)`. 这种浅薄使`HAIR`表示所有数据类型[希尔]有，但不必创建整个[希尔]. [密尔]降低将首先从[希尔]到[头发]中[rustc_mir：：hair：：cx：：expr::]）然后处理[头发]递归表达式。

降低会为签名中指定的每个参数创建局部变量。接下来，它为每个指定的绑定创建局部变量（例如`(a, b): (i32, String)`）生成3个绑定，一个用于参数，两个用于绑定。接下来，它生成字段访问，从参数中读取字段并将值写入绑定变量。

在这种初始化过程中，降低操作将触发对为主体生成mir的函数的递归调用（a`Block`表达式）并将结果写入`RETURN_PLACE`.

## `unpack!`所有的事情

生成mir的函数倾向于分为两种模式之一。首先，如果函数只生成语句，那么它将使用一个基本块作为参数，将这些语句附加到该参数上。然后它可以正常返回结果：

```rust,ignore
fn generate_some_mir(&mut self, block: BasicBlock) -> ResultType {
   ...
}
```

但也有其他函数可以生成新的基本块。例如，降低表达式`if foo { 22 } else { 44 }`需要生成一个小的“菱形图形”。在这种情况下，函数在其代码开始处接受一个基本块，并在代码生成结束处返回一个（可能）新的基本块。这个`BlockAnd`类型用于表示此：

```rust,ignore
fn generate_more_mir(&mut self, block: BasicBlock) -> BlockAnd<ResultType> {
    ...
}
```

当调用这些函数时，通常使用局部变量`block`这实际上是一个“光标”。它代表了我们添加新mir的点。当你调用`generate_more_mir`，要更新此光标。您可以手动执行此操作，但这很乏味：

```rust,ignore
let mut block;
let v = match self.generate_more_mir(..) {
    BlockAnd { block: new_block, value: v } => {
        block = new_block;
        v
    }
};
```

因此，我们提供了一个宏，允许您编写`let v = unpack!(block = self.generate_more_mir(...))`. 它只提取新块并覆盖变量`block`你在`unpack!`.

## 将表达式降低到所需的mir中

基本上有四种表示形式，一种可能需要表达式：

-   `Place`指预先存在的内存位置（本地、静态、提升）
-   `Rvalue`是可以分配给`Place`
-   `Operand`是例如`+`操作或函数调用
-   包含值副本的临时变量

下图概述了表示之间的交互作用：

<img src="mir_overview.svg">

[单击此处查看更详细的视图](mir_detailed.svg)

我们从降低功能体到`Rvalue`因此我们可以创建一个分配给`RETURN_PLACE`，这个`Rvalue`降低将依次触发降低至`Operand`因为它的论点（如果有的话）。`Operand`降低任何一个都会产生`const`操作数，或从`Place`从而触发`Place`降低。将表达式降为`Place`如果要降低的表达式包含操作，则会触发要创建的临时。这就是蛇咬住自己尾巴的地方，我们需要触发`Rvalue`降低以将表达式写入本地。

## 操作员下降

内置类型上的运算符不会降低为函数调用（最终将成为无限递归调用，因为特征impl只包含操作本身）。取而代之的是`Rvalue`用于二进制和一元运算符以及索引操作。这些`Rvalue`然后将代码生成为llvm基元操作或llvm内部函数。

所有其他类型的运算符都将降低为对其`impl`操作员的相应特征。

无论降低类型如何，运算符的参数都将降低到`Operand`这意味着所有参数都是常量，或者引用本地或静态中已经存在的值。

## 方法调用降低

方法调用被降低到相同的`TerminatorKind`函数调用是。在[密尔]方法调用和函数调用之间已经没有区别了。

## 条件

`if`条件和`match`声明`enum`不带字段变量的s被降低到`TerminatorKind::SwitchInt`.每个可能的值（所以`0`和`1`对于`if`条件）具有相应的`BasicBlock`代码将继续执行。正在进行分支的论点是（再次）一个`Operand`表示if条件的值。

### 模式匹配

`match`声明`enum`具有字段的变量的将降低到`TerminatorKind::SwitchInt`也一样，但是`Operand`指的是`Place`在这里可以找到值的判别式。这通常涉及到将判别式读取到一个新的临时变量。

## 集料施工

任何类型的聚合值（例如结构或元组）都是通过`Rvalue::Aggregate`.所有字段都降低到`Operator`这实质上相当于每个聚合字段一个赋值语句加上在下列情况下对判别式的赋值`enum`s.

[mir]: ./index.html

[hir]: ../hir.html

[hair]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/hair/index.html

[rustc_mir::hair::cx::expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/hair/cx/expr/index.html

[`mir_built`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/fn.mir_built.html
