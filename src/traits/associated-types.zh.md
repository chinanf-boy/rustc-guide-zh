# 相等和关联类型

本节介绍特征系统如何处理关联类型之间的相等性。整个系统由几个运动部件组成，我们将逐一介绍：

-   投影和`Normalize`谓语
-   与占位符关联的类型投影
-   这个`ProjectionEq`谓语
-   与统一融合

## 关联类型投影和规范化

当一个特征定义了一个相关的类型（例如，[这个`Item`键入`IntoIterator`特质][intoiter-item]，用户可以使用**关联类型投影**喜欢`<Option<u32> as IntoIterator>::Item`.

> 通常，人们会使用速记语法`T::Item`. 目前，该语法在[“类型集合”](../type-checking.html)变成明确的形式，尽管这是我们将来可能想要改变的。

[intoiter-item]: https://doc.rust-lang.org/nightly/core/iter/trait.IntoIterator.html#associatedtype.Item

<a name="normalize"></a>

在某些情况下，关联的类型投影可以是**归一化**—也就是说，简化了—基于IMPL中给出的类型。因此，为了继续我们的示例，`IntoIterator`对于`Option<T>`宣布`Item = T`：

```rust,ignore
impl<T> IntoIterator for Option<T> {
  type Item = T;
  ...
}
```

这意味着我们可以规范化投影`<Option<u32> as
IntoIterator>::Item`只是`u32`.

在这种情况下，投影是一个“单态”投影——也就是说，它没有任何类型参数。单态投影是特殊的，因为它们可以**总是**完全标准化。

通常，我们也可以规范化其他相关类型的投影。例如，`<Option<?T> as IntoIterator>::Item`在哪里`?T`是一个推理变量，可以规范化为`?T`.

在我们的逻辑中，规范化是由谓词定义的`Normalize`. 这个`Normalize`条款仅由IMPLS产生。例如，`impl`属于`IntoIterator`对于`Option<T>`我们在上面看到的将被降为类似这样的程序子句：

```text
forall<T> {
    Normalize(<Option<T> as IntoIterator>::Item -> T) :-
        Implemented(Option<T>: IntoIterator)
}
```

在这种情况下，这个`Implemented`条件总是正确的。

> 因为我们不允许对特性进行量化，所以这更像是一系列程序子句，每个相关类型对应一个。

我们可以应用这个规则来规范化我们迄今为止看到的任何一个例子。

## 占位符关联类型

但是，有时我们希望处理不能规范化的关联类型。例如，考虑这个函数：

```rust,ignore
fn foo<T: IntoIterator>(...) { ... }
```

在这个上下文中，我们如何规范化类型`T::Item`？

不知道什么`T`是的，我们不能真的这么做。为了表示这种情况，我们引入一个称为**占位符关联类型投影**. 这是这样写的：`(IntoIterator::Item)<T>`.

您可能会注意到，它看起来非常像常规类型（例如，`Option<T>`，但类型的“名称”是`(IntoIterator::Item)`. 这不是意外：占位符关联的类型投影与普通类型类似`Vec<T>`当谈到统一时。也就是说，只有当（a）它们都是对同一关联类型的引用，例如`IntoIterator::Item`和（b）它们的类型参数相等。

占位符关联类型从不由用户直接写入。它们只在特性系统内部使用，稍后我们将看到。

在Rustc中，它们对应于`TyKind::UnnormalizedProjectionTy`枚举变量，在中声明[`librustc/ty/sty.rs`][sty]. 我们用粉笔`ApplicationTy`名称位于专用于占位符关联类型的特殊命名空间中（请参见`TypeName`枚举声明于[`chalk-ir/src/lib.rs`][chalk_type_name]）

[sty]: https://github.com/rust-lang/rust/blob/master/src/librustc/ty/sty.rs

[chalk_type_name]: https://github.com/rust-lang-nursery/chalk/blob/master/chalk-ir/src/lib.rs

## 投影相等

到目前为止，我们已经看到了两种方法来回答这个问题：“我们何时可以考虑关联类型投影等于另一种类型？”：

-   这个`Normalize`当我们知道哪个impl适用时，谓词可以用来转换投影；
-   **占位符**当我们不使用时，可以使用关联的类型。这也被称为**惰性规范化**.

我们现在介绍`ProjectionEq`谓词将这两个事例组合在一起。这个`ProjectionEq`谓词如下：

```text
ProjectionEq(<T as IntoIterator>::Item = U)
```

我们将看到它可以被证明*任何一个*通过规范化或通过占位符类型。作为从某个特性降低关联类型声明的一部分，我们为`ProjectionEq`：

```text
forall<T, U> {
    ProjectionEq(<T as IntoIterator>::Item = U) :-
        Normalize(<T as IntoIterator>::Item -> U)
}

forall<T> {
    ProjectionEq(<T as IntoIterator>::Item = (IntoIterator::Item)<T>)
}
```

只有这两个`ProjectionEq`我们为任何给定的关联项所做的程序条款。

## 与统一融合

现在我们准备讨论关联类型平等如何与统一相结合。如中所述[类型推断](../type-inference.html)第节，统一基本上是一个带有如下签名的过程：

```text
Unify(A, B) = Result<(Subgoals, RegionConstraints), NoSolution>
```

换句话说，我们试图把A和B两个东西统一起来，这个过程可能会失败，在这种情况下，我们会回来的。`Err(NoSolution)`. 例如，如果我们试图统一`u32`和`i32`.

关键是，在成功的时候，统一也可以给我们一组有待证明的子目标。（它也可以返回区域约束，但这些在这里不相关）。

每当统一遇到与非占位符关联的类型投影P与其他类型T相等时，它总是成功的，但它生成子目标。`ProjectionEq(P = T)`这是传播备份。因此，处理这一约束属于特征系统的普通工作。

> 如果我们统一两个投影p1和p2，那么统一会产生一个变量x，并要求我们证明`ProjectionEq(P1 = X)`和`ProjectionEq(P2 = X)`. （以前在旧系统中，为了防止循环，这是必须的；我宁愿怀疑它仍然存在。- NMASSAKIS
