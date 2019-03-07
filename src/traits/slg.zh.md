# 按需SLG解决方案

提供一套程序条款（由我们提供[下沉规则][lowering]）对于查询，我们需要返回查询的结果和我们可以确定的任何类型变量的值。这是解算器的工作。

例如，`exists<T> { Vec<T>: FromIterator<u32> }`有一个解决方案，因此其结果是`Unique; substitution [?T := u32]`. 解决方案还附带了一组区域约束，我们将在本介绍中忽略这些约束。

[lowering]: ./lowering-rules.html

## 解决者的目标

### 按需

查询的解决方案通常很多，甚至无穷多。例如，假设我们想证明`exists<T> { Vec<T>: Debug }`对于*一些*类型`?T`. 我们的解决方案应该能够一次给出一个答案，比如`?T = u32`然后`?T = i32`，等等，而不是对类型系统中的每个类型进行迭代。如果我们需要更多的答案，我们可以要求更多，直到我们完成。这类似于Prolog的工作方式。

*参见：[传统的交互式Prolog查询][pq]*

[pq]: ./canonical-queries.html#the-traditional-interactive-prolog-query

### 宽度优先

`Vec<?T>: Debug`如果是真`?T: Debug`. 这会导致一个循环：`[Vec<u32>,
Vec<Vec<u32>>, Vec<Vec<Vec<u32>>>]`等等，所有的工具`Debug`. 我们的解决方案应该是广度优先，并考虑以下答案`[Vec<u32>: Debug,
Vec<i32>: Debug, ...]`否则我们可能永远找不到我们要找的答案。

### 可缓存的

为了加快编译速度，我们需要缓存结果，包括以前的规划求解查询留下的部分结果。

## 如何工作的描述

解算器的基础是[`Forest`]类型。一*森林*存储的集合*桌子*以及A*堆栈*. 各*桌子*表示正在执行的特定查询的存储结果，以及*股线*这基本上是暂停计算，可以用来寻找更多的答案。表是相互依赖的：解决一个查询可能需要解决其他查询。

[`forest`]: https://rust-lang-nursery.github.io/chalk/doc/chalk_engine/forest/struct.Forest.html

### 演练

也许解释解算器如何工作的最简单方法是通过一个例子。假设我们有以下程序：

```rust,ignore
trait Debug { }

struct u32 { }
impl Debug for u32 { }

struct Rc<T> { }
impl<T: Debug> Debug for Rc<T> { }

struct Vec<T> { }
impl<T: Debug> Debug for Vec<T> { }
```

现在假设我们想找到这个问题的答案`exists<T> { Rc<T>:
Debug }`. 第一步是u-规范化这个查询；这是根据所有未绑定的推理变量最左边出现的顺序给它们指定规范化名称的行为，以及规范化任何通用绑定名称的universe（例如`T`在里面`forall<T> { ...
}`）在这种情况下，没有普遍绑定的名称，但查询的规范形式q可能类似于：

```text
Rc<?0>: Debug
```

在哪里？`?0`是根宇宙U0中的变量。然后，我们将查找一个以该规范查询为键的表：由于林是空的，因此查找将失败，我们将创建一个与U规范目标Q对应的新表t0。

**忽略负面推理和区域。**首先，我们会忽略消极目标的可能性，比如`not { Foo }`. 我们稍后会分阶段实施，因为它们会带来一些并发症。

**创建表。**当我们第一次创建表时，我们还用一组*初始股线*. “流”有点像求解器的“线程”：它包含生成答案的特定方法。一个目标的初始系列`Rc<?0>: Debug`（即“领域目标”）通过寻找*条款*在环境中。在Rust中，这些条款源于IMPLS，也源于范围内的WHERE条款。在我们的例子中，有三个从句，每个从句来自程序。使用一个类似prolog的符号，这些符号看起来像：

```text
(u32: Debug).
(Rc<T>: Debug) :- (T: Debug).
(Vec<T>: Debug) :- (T: Debug).
```

为了创建我们的初始系列，那么，我们将尝试将这些条款中的每一个应用到我们的目标`Rc<?0>: Debug`.第一和第三条不适用，因为`u32`和`Vec<?0>`无法与统一`Rc<?0>`.然而，第二条是可行的。

**什么是股？**让我们多谈一谈什么是股线*是*.在代码中，链是推理表和*X条款*和（可能）从该X子句中选择的子目标。但是什么是X条款（[`ExClause`]，在代码中）？一个x子句将一些东西组合在一起：

-   我们试图证明的目标的当前状态；
-   一组尚未被证明的子目标；
-   我们现在还忽略了一些事情：
    -   延迟文本，区域约束

x子句的一般形式与prolog子句非常相似，但语义有所不同。因为我们忽略了延迟的文本和区域约束，所以一个x子句如下所示：

```text
G :- L
```

其中g是一个目标，l是一组必须被证明的子目标。（L代表*字面意义的*--当我们讨论否定推理时，字面上的内容要么是肯定的子目标，要么是否定的子目标。）我们的想法是，如果我们能够证明l，那么目标g可以被认为是真的。

在我们的示例中，我们最终将创建一个链，使用这样的x子句：

```text
(Rc<?T>: Debug) :- (?T: Debug)
```

这里，`?T`是指在推理表中创建的与链一起的推理变量之一。（我将使用命名变量来引用推理变量，以及编号变量，如`?0`引用规范化目标中的变量；然而，在代码中，它们都用索引表示。）

对于每个链，我们还可以选择存储*选定的子目标*. 这是旋转栅门后的子目标（`:-`）我们目前正在努力证明这一点。最初，当第一次创建链时，没有选定的子目标。

[`exclause`]: https://rust-lang-nursery.github.io/chalk/doc/chalk_engine/struct.ExClause.html

**激活股线。**既然我们已经创建了表t0并用链对其进行了初始化，那么我们必须实际尝试并生成一个答案。我们通过调用[`ensure_root_answer`]手术台上的操作：具体来说，我们说`ensure_root_answer(T0, A0)`，表示“确保有第0个回答a0查询t0”。

记住，表不仅存储链，还存储缓存答案的向量。第一件事是[`ensure_root_answer`]做的是检查答案a0是否在这个向量中。如果是的话，我们可以马上回来。在这种情况下，向量将为空，因此不适用（这对于以后的循环检查很重要）。

当没有缓存的答案时，[`ensure_root_answer`]将尝试生产一个。它通过从一组活动股中选择一股来实现这一点——股存储在`VecDeque`因此以循环方式处理。现在，我们只有一个链，存储以下没有选定子目标的x子句：

```text
(Rc<?T>: Debug) :- (?T: Debug)
```

当我们激活链时，我们看到我们没有选择的子目标，因此我们首先选择要处理的子目标之一。这里，只有一个（`?T: Debug`，使其成为选定的子目标，将链的状态更改为：

```text
(Rc<?T>: Debug) :- selected(?T: Debug, A0)
```

在这里，我们写`selected(L, An)`表示（a）文字`L`所选子目标和（b）哪个答案`An`我们正在寻找。我们开始寻找`A0`.

[`ensure_root_answer`]: https://rust-lang-nursery.github.io/chalk/doc/chalk_engine/forest/struct.Forest.html#method.ensure_root_answer

**正在处理选定的子目标。**接下来，我们必须尝试找到这个选定目标的答案。为此，我们将U-规范化它，并尝试查找关联表。在这种情况下，子目标的U规范形式是`?0: Debug`：我们还没有这个表，所以我们可以创建一个新的表T1。和以前一样，我们将用线股初始化T1。在这种情况下，将有三条线，因为所有程序条款都可能适用。这三股将是：

-   `(u32: Debug) :-`，从PROGRAM子句派生`(u32: Debug).`.
    -   注意：这个链没有子目标。
-   `(Vec<?U>: Debug) :- (?U: Debug)`，源自`Vec`IMPL
-   `(Rc<?U>: Debug) :- (?U: Debug)`，源自`Rc`IMPL

因此，我们可以将整个森林在这一点上的状态总结如下：

```text
Table T0 [Rc<?0>: Debug]
  Strands:
    (Rc<?T>: Debug) :- selected(?T: Debug, A0)
  
Table T1 [?0: Debug]
  Strands:
    (u32: Debug) :-
    (Vec<?U>: Debug) :- (?U: Debug)
    (Rc<?V>: Debug) :- (?V: Debug)
```

**表之间的委托。**现在，来自t0的活动链已经创建了表T1，它可以尝试提取一个答案。它也是通过这个`ensure_answer`我们以前看过的手术。在这种情况下，流将调用`ensure_answer(T1, A0)`，因为我们将从第一个答案开始。这将导致T1激活其第一股，`u32: Debug :-`.

这条线有点特别：它根本没有子目标。这意味着目标已经被证明。因此我们可以补充`u32: Debug`到*答案*对于我们的桌子，称之为回答a0（这是第一个答案）。然后将钢绞线从钢绞线列表中删除。

因此，表T1的状态为：

```text
Table T1 [?0: Debug]
  Answers:
    A0 = [?0 = u32]
  Strand:
    (Vec<?U>: Debug) :- (?U: Debug)
    (Rc<?V>: Debug) :- (?V: Debug)
```

请注意，我写的答案a0是一个可以应用于表目标的替换；实际上，在代码中，每个x子句的目标也表示为替换，但在本文中，我选择将它们作为完整的目标来写，如下所示[NFTD公司].

[nftd]: ./bibliography.html#slg

既然我们现在有了答案，`ensure_answer(T1, A0)`会回来的`Ok`到表t0，表示答案a0可用。现在，t0的任务是将结果合并到其活动链中。这有两种方式。首先，它创建了一个新的链，寻找T1的下一个可能的答案。接下来，它将a0的答案合并在一起，并删除子目标。表t0的结果状态为：

```text
Table T0 [Rc<?0>: Debug]
  Strands:
    (Rc<?T>: Debug) :- selected(?T: Debug, A1)
    (Rc<u32>: Debug) :-
```

然后我们立即激活包含答案的链`Rc<u32>: Debug`一）。在这种情况下，该链没有进一步的子目标，因此它成为表t0的答案。然后，这个答案可以返回给我们的调用者，此时整个森林都会静止不动（记住，我们只做了足够的工作来生成*一*回答）此时林的结束状态为：

```text
Table T0 [Rc<?0>: Debug]
  Answer:
    A0 = [?0 = u32]
  Strands:
    (Rc<?T>: Debug) :- selected(?T: Debug, A1)

Table T1 [?0: Debug]
  Answers:
    A0 = [?0 = u32]
  Strand:
    (Vec<?U>: Debug) :- (?U: Debug)
    (Rc<?V>: Debug) :- (?V: Debug)
```

在这里，您可以看到森林是如何捕获到目前为止我们创建的两个答案的。*和*让我们稍后尝试产生更多答案的线索。

## 也见

-   [粉笔解决自述][readme]其中包含到所用论文的链接和代码中引用的缩写词
-   这一部分是对博客文章稍加修改的版本。[粉笔的按需SLG解算器][slg-blog]
-   [粉笔中的否定推理][negative-reasoning-blog]解释否定推理的必要性，但不解释SLG解算器是如何做到的。

[readme]: https://github.com/rust-lang-nursery/chalk/blob/239e4ae4e69b2785b5f99e0f2b41fc16b0b4e65e/chalk-engine/src/README.md

[slg-blog]: http://smallcultfollowing.com/babysteps/blog/2018/01/31/an-on-demand-slg-solver-for-chalk/

[negative-reasoning-blog]: http://aturon.github.io/blog/2017/04/24/negative-chalk/
