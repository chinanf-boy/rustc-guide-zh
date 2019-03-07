# 规范查询

特征系统的“开始”是**规范查询**（这些都是更一般意义上的疑问——你想知道答案的东西——以及[Rustc特定意义](../query.html)）其思想是，类型检查器或系统的其他部分在执行任务的过程中可能希望知道某个特性是否为某个类型实现（例如，是`u32: Debug`真的？）或者他们可能想[规范化某些关联类型](./associated-types.html).

本节将在相当高的抽象级别上讨论查询。各小节更仔细地研究了这些思想是如何在RustC中实现的。

## 传统的交互式Prolog查询

在传统的prolog系统中，当您开始查询时，解算器将运行并开始向您提供它能找到的所有可能的答案。所以有了这样的条件：

```text
?- Vec<i32>: AsRef<?U>
```

解算器可能会回答：

```text
Vec<i32>: AsRef<[i32]>
    continue? (y/n)
```

这个`continue`有点有趣。Prolog中的想法是解算器正在查找**一切可能**查询的实例化为真。在这种情况下，如果我们实例化`?U = [i32]`，那么查询是真的（注意，传统的prolog接口不会直接告诉我们`?U`但是，我们可以通过将响应与原始查询统一起来推断出一个结果——Rust的解算器返回替换结果）。如果我们要打`y`然后，解算器可能会给我们另一个可能的答案：

```text
Vec<i32>: AsRef<Vec<i32>>
    continue? (y/n)
```

这个答案来自这样一个事实，即存在一个自反的implexive（`impl<T> AsRef<T> for T`为`AsRef`. 如果命中`y`同样，我们可能会得到一个否定的回答：

```text
no
```

自然地，在某些情况下，可能没有可能的答案，因此解决者只会把我还给我。`no`马上：

```text
?- Box<i32>: Copy
    no
```

在某些情况下，可能有无限多的响应。例如，如果我给出了这个查询，并且不断地点击`y`，那么解算器就永远不会停止给我答案：

```text
?- Vec<?U>: Clone
    Vec<i32>: Clone
        continue? (y/n)
    Vec<Box<i32>>: Clone
        continue? (y/n)
    Vec<Box<Box<i32>>>: Clone
        continue? (y/n)
    Vec<Box<Box<Box<i32>>>>: Clone
        continue? (y/n)
```

如你所能想象的，解算器会高兴地继续添加另一层`Box`直到我们要求它停止，或者它耗尽了记忆。

另一个有趣的事情是查询中可能还有变量。例如：

```text
?- Rc<?T>: Clone
```

可能会得出答案：

```text
Rc<?T>: Clone
    continue? (y/n)
```

毕竟，`Rc<?T>`是真的**不管是什么类型的`?T`是**.

<a name="query-response"></a>

## Rustc中的特征查询

Rustc中的特征查询工作方式有所不同。而不是试图列举**一切可能**你的答案，他们正在寻找**明确的**回答。特别是，当它们告诉您类型变量的值时，这意味着这是**只有可能的实例化**如果给定当前的IMPLS集合和WHERE子句，您可以使用它，这是可以证明的。（不过，在求解器内部，它们可能会枚举所有可能的答案。见[SLG解算器的描述](./slg.html)详情。

对rustc中的特征查询的响应通常是`Result<QueryResult<T>, NoSolution>`（在那里`T`根据查询本身的不同而有所不同）。这个`Err(NoSolution)`case表示查询是假的，没有答案（例如，`Box<i32>: Copy`）否则，`QueryResult`提供有关我们找到的可能答案的信息。它由四部分组成：

-   **确定性：**告诉你我们对这个答案有多确定。它可以有两个值：
    -   `Proven`意味着结果是真实的。
        -   这可能是试图证明的结果`Vec<i32>: Clone`比如说`Rc<?T>: Clone`.
    -   `Ambiguous`意味着有些事情我们还不能证明是真的*或*错误，通常是因为需要更多类型信息。（稍后我们将看到一个示例。）
        -   这可能是试图证明的结果`Vec<?T>: Clone`.
-   **VaR值：**每个未绑定推理变量的值（如`?T`）出现在原始查询中。（记住，在Prolog中，我们必须推断出这些。）
    -   正如我们在下面的示例中看到的，我们甚至可以为`Ambiguous`病例。
-   **区域约束：**这些关系必须在作为输入提供的生命周期之间保持。我们将忽略这些，但请看[性状处理区](./regions.html)了解更多详细信息。
-   **价值观：**查询结果还带有类型为的值`T`. 对于一些专门的查询（如规范化关联类型），这用于返回一个额外的结果，但通常只是`()`.

### 实例

让我们通过一个示例查询来了解所有部分的含义。考虑[这个`Borrow`特质][borrow]. 这个特性有许多impl；其中有两个（为了清楚起见，我已经写了`Sized`明确界限）：

[borrow]: https://doc.rust-lang.org/std/borrow/trait.Borrow.html

```rust,ignore
impl<T> Borrow<T> for T where T: ?Sized
impl<T> Borrow<[T]> for Vec<T> where T: Sized
```

**例1。**假设我们正在检查这个（相当人工的）代码位：

```rust,ignore
fn foo<A, B>(a: A, vec_b: Option<B>) where A: Borrow<B> { }

fn main() {
    let mut t: Vec<_> = vec![]; // Type: Vec<?T>
    let mut u: Option<_> = None; // Type: Option<?U>
    foo(t, u); // Example 1: requires `Vec<?T>: Borrow<?U>`
    ...
}
```

如注释所示，我们首先创建两个变量`t`和`u`；`t`是一个空向量，并且`u`是一个`None`选择权。这两个变量在其类型中都有未绑定的推理变量：`?T`表示矢量中的元素`t`和`?U`表示存储在选项中的值`u`.  接下来，我们调用`foo`；比较的签名`foo`根据它的论点，我们最终`A = Vec<?T>`和`B = ?U`因此，关于`foo`要求`Vec<?T>:
Borrow<?U>`. 因此，这是我们的第一个示例特征查询。

查询有许多可能的解决方案`Vec<?T>: Borrow<?U>`例如：

-   `?U = Vec<?T>`，
-   `?U = [?T]`，
-   `?T = u32, ?U = [u32]`
-   诸如此类。

因此，我们得到的结果如下（我将忽略区域约束和“值”）：

-   确定性：`Ambiguous`–我们还不确定这是否适用。
-   VaR值：`[?T = ?T, ?U = ?U]`–我们对变量的值一无所知。

简言之，查询结果表明，现在谈论这个特性是否被证明还为时过早。在类型检查期间，这不是一个即时错误：相反，类型检查程序将保留此要求。（`Vec<?T>: Borrow<?U>`等等。正如我们在下一个示例中看到的，可能会发生这种情况`?T`和`?U`最终会受到其他来源的约束，在这种情况下，我们可以再次尝试特征查询。

**例2。**现在我们可以稍微扩展前面的示例，并将值赋给`u`：

```rust,ignore
fn foo<A, B>(a: A, vec_b: Option<B>) where A: Borrow<B> { }

fn main() {
    // What we saw before:
    let mut t: Vec<_> = vec![]; // Type: Vec<?T>
    let mut u: Option<_> = None; // Type: Option<?U>
    foo(t, u); // `Vec<?T>: Borrow<?U>` => ambiguous

    // New stuff:
    u = Some(vec![]); // ?U = Vec<?V>
}
```

由于这项任务，类型`u`被迫成为`Option<Vec<?V>>`在哪里`?V`表示矢量的元素类型。这反过来意味着`?U`是[统一的]到`Vec<?V>`.

[unified]: ../type-checking.html

假设类型检查器决定重新检查我们以前看到的“尚未验证”的特性义务，`Vec<?T>:
Borrow<?U>`. `?U`不再是未绑定的推理变量；它现在有一个值，`Vec<?V>`. 因此，如果使用该值“刷新”查询，我们将得到：

```text
Vec<?T>: Borrow<Vec<?V>>
```

这一次，只有一个IMPL适用，即反射IMPL：

```text
impl<T> Borrow<T> for T where T: ?Sized
```

因此，特征检查者将回答：

-   确定性：`Proven`
-   VaR值：`[?T = ?T, ?V = ?T]`

在这里，它是说，我们确实已经证明了义务是成立的，而且我们也知道`?T`和`?V`是相同的类型（但我们还不知道该类型是什么！）.

（事实上，当函数在这里结束时，类型检查器此时会给出一个错误，因为`t`和`u`尽管已知它们是相同的，但仍不为人所知。）
