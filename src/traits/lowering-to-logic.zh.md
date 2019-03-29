# 降低到逻辑

这里的关键观察是，Rust 特征系统基本上是一种逻辑，它可以映射到标准的逻辑推理规则上。然后，我们可以用非常类似的方式寻找这些推理规则的解决方案，例如[序言]解算器工作。结果我们不能*相当地*使用 prolog 规则（也称为 horn 子句），但需要更具表达力的变体。

[prolog]: https://en.wikipedia.org/wiki/Prolog

## Rust 特征与逻辑

最初的观察之一是，Rust 特征系统基本上是一种逻辑。因此，我们可以将结构、特性和 IMPL 声明映射到逻辑推理规则中。在大多数情况下，这些基本上都是 horn 子句，尽管我们将看到，为了获得充分的信任（尤其是支持通用编程），我们必须比标准 horn 子句更进一步。

为了了解这个映射是如何工作的，让我们从一个例子开始。假设我们声明了一个特征和一些简单的例子，比如：

```rust
trait Clone { }
impl Clone for usize { }
impl<T> Clone for Vec<T> where T: Clone { }
```

我们可以将这些声明映射到一些用类似序言的符号编写的 Horn 子句，如下所示：

```text
Clone(usize).
Clone(Vec<?T>) :- Clone(?T).

// The notation `A :- B` means "A is true if B is true".
// Or, put another way, B implies A.
```

在 prolog 术语中，我们可以这样说`Clone(Foo)`—在哪里`Foo`是铁锈型的-是*谓语*它代表了`Foo`器具`Clone`. 这些规则是**纲领性条款**；它们说明可以证明该谓词的条件（即，认为是真的）。所以第一条规则只是说“克隆是为`usize`“。下一条规则说“任何类型`?T`，为实现克隆`Vec<?T>`如果为实现克隆`?T`“。例如，如果我们想证明`Clone(Vec<Vec<usize>>)`，我们将通过递归应用规则来实现这一点：

- `Clone(Vec<Vec<usize>>)`可证明：
  - `Clone(Vec<usize>)`可证明：
    - `Clone(usize)`是可以证明的。（就是这样，所以我们都很好。）

但是现在假设我们试图证明`Clone(Vec<Bar>)`. 这会失败的（毕竟，我没有暗示`Clone`对于`Bar`）：

- `Clone(Vec<Bar>)`可证明：
  - `Clone(Bar)`是可以证明的。（但事实并非如此，因为没有适用的规则。）

我们可以很容易地扩展上面的示例，以涵盖具有多个输入类型的通用特性。所以想象一下`Eq<T>`特征，这说明`Self`与类型的值相等`T`：

```rust,ignore
trait Eq<T> { ... }
impl Eq<usize> for usize { }
impl<T: Eq<U>> Eq<Vec<U>> for Vec<T> { }
```

可以映射如下：

```text
Eq(usize, usize).
Eq(Vec<?T>, Vec<?U>) :- Eq(?T, ?U).
```

到现在为止，一直都还不错。

## 类型检查正常功能

好吧，既然我们已经定义了一些逻辑规则，这些规则能够在实现特性和处理相关类型时表示出来，那么让我们将重点转向**类型检查**. 类型检查很有趣，因为它给了我们需要证明的目标。也就是说，到目前为止，我们所看到的一切都是关于我们如何根据程序的特点和隐含来证明目标的规则；但是我们也对如何得出我们需要证明的目标感兴趣，那些目标来自类型检查。

考虑类型检查函数`foo()`在这里：

```rust,ignore
fn foo() { bar::<usize>() }
fn bar<U: Eq<U>>() { }
```

当然，这个函数非常简单：它所做的就是调用`bar::<usize>()`. 现在，看看`bar()`，我们可以看到它有一个 WHERE 子句`U: Eq<U>`. 所以，这意味着`foo()`必须证明`usize: Eq<usize>`为了显示它可以调用`bar()`具有`usize`作为类型参数。

如果需要的话，我们可以编写一个 prolog 谓词来定义在什么条件下`bar()`可以称之为。我们会说这些条件被称为“形成良好的”条件：

```text
barWellFormed(?U) :- Eq(?U, ?U).
```

那么我们可以这么说`foo()`类型检查引用是否`bar::<usize>`（也就是说，`bar()`应用于类型`usize`）形状良好：

```text
fooTypeChecks :- barWellFormed(usize).
```

如果我们试图证明这个目标`fooTypeChecks`，它将成功：

- `fooTypeChecks`可证明：
  - `barWellFormed(usize)`，如果：
    - `Eq(usize, usize)`这是可以证明的，因为一个 IMPL。

好的，到目前为止还不错。让我们继续类型检查一个更复杂的函数。

## 类型检查通用功能：超出喇叭条款

在最后一节中，我们使用标准的 prolog horn 子句（在 rust 的类型相等概念的基础上增加）来对一些简单的 rust 函数进行类型检查。但这只在我们检查非泛型函数的类型时有效。如果我们想键入 check 一个泛型函数，结果是我们需要一个比 prolog 能够提供的更强大的目标概念。为了了解我所说的内容，我们将前面的示例修改为`foo`通用的：

```rust,ignore
fn foo<T: Eq<T>>() { bar::<T>() }
fn bar<U: Eq<U>>() { }
```

键入检查的正文`foo`，我们需要能够保存类型`T`“抽象”。也就是说，我们需要检查`foo`类型安全*适用于所有类型`T`*，不只是针对某些特定类型。我们可以这样表达：

```text
fooTypeChecks :-
  // for all types T...
  forall<T> {
    // ...if we assume that Eq(T, T) is provable...
    if (Eq(T, T)) {
      // ...then we can prove that `barWellFormed(T)` holds.
      barWellFormed(T)
    }
  }.
```

我在这里使用的这个符号是我在原型实现中使用的符号；它类似于标准的数学符号，但有点生锈。不管怎样，问题是标准 Horn 条款不允许通用量化（`forall`）或暗示（`if`）在目标中（尽管许多 prolog 引擎确实支持它们，作为扩展）。出于这个原因，我们需要接受一个叫做“一阶遗传哈罗普”（fohh）条款的东西——这个长名称基本上是指“标准喇叭条款`forall`和`if`“在体内”。但是知道正确的名称是很好的，因为有很多工作描述如何有效地处理 FOHH 子句；例如，参见 Gopalan Nadathur's Excellent[“遗传哈罗普公式逻辑的证明程序”][pphhf]在里面[参考文献].

[the bibliography]: ./bibliography.html
[pphhf]: ./bibliography.html#pphhf

事实证明，支持 Fohh 并不是那么难。一旦我们能够做到这一点，我们就可以很容易地描述类泛型函数的类型检查规则`foo`在我们的逻辑中。

## 来源

本页是对[Nicholas Matsakis 的博客帖子][lrtl].

[lrtl]: http://smallcultfollowing.com/babysteps/blog/2017/01/26/lowering-rust-traits-to-logic/
