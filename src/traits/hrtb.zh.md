# 高等级性状界限

特征分辨的一个更微妙的概念是*高等级性状界限*.这种界限的一个例子是`for<'a> MyTrait<&'a isize>`.让我们来看看高等级特征参照的选择是如何工作的。

## 基本匹配和占位符泄漏

假设我们有一个特点`Foo`以下内容：

```rust
trait Foo<X> {
    fn foo(&self, x: X) { }
}
```

假设我们有一个函数`want_hrtb`需要一个实现`Foo<&'a isize>`对于任何`'a`以下内容：

```rust,ignore
fn want_hrtb<T>() where T : for<'a> Foo<&'a isize> { ... }
```

现在我们有了一个结构`AnyInt`实现`Foo<&'a isize>`对于任何`'a`以下内容：

```rust,ignore
struct AnyInt;
impl<'a> Foo<&'a isize> for AnyInt { }
```

问题是，是吗？`AnyInt : for<'a> Foo<&'a isize>`是吗？我们希望答案是肯定的。计算出它的算法与高排名类型的子类型（如所述）密切相关[在这里][hrsubtype]也在一个[SPJ论文].如果您想了解更高级别的子类型，我们建议您阅读本文）。有几个部分：

1.  用占位符替换债务中的绑定区域。
2.  将IMPL与[占位符]义务。
3.  检查*占位符泄漏*.

[placeholder]: ../appendix/glossary.html#appendix-c-glossary

[hrsubtype]: https://github.com/rust-lang/rust/tree/master/src/librustc/infer/higher_ranked/README.md

[paper by spj]: http://research.microsoft.com/en-us/um/people/simonpj/papers/higher-rank/

所以让我们来研究一下我们的例子。

1.  我们要做的第一件事是用占位符替换义务中的绑定区域，从而`AnyInt : Foo<&'0 isize>`（这里）`'0`表示占位符区域0）。注意，我们现在没有量词；就编译器类型而言，这是从`ty::PolyTraitRef`到A`TraitRef`. 然后我们将创建`TraitRef`从IMPL中，为其绑定区域使用新变量（从而`Foo<&'$a isize>`在哪里`'$a`是的推理变量`'a`）

2.  接下来，我们将这两个特征引用联系起来，生成一个带有约束的图。`'0 == '$a`.

3.  最后，我们检查占位符“泄漏”-泄漏基本上是指试图将占位符区域与另一个占位符区域或之前存在IMPL匹配的任何区域关联起来的任何尝试。泄漏检查是通过从占位符区域中搜索以查找与之相关的区域集来完成的。这就是所谓的“污点”集合。要通过检查，该集合必须包含*唯一地*来自IMPL的自身变量和区域变量。如果污点集包含任何其他区域，则匹配失败。在这种情况下，污点`'0`是`{'0, '$a}`，因此检查将成功。

让我们考虑一个失败案例。假设我们还有一个结构

```rust,ignore
struct StaticInt;
impl Foo<&'static isize> for StaticInt;
```

我们想要义务`StaticInt : for<'a> Foo<&'a isize>`被认为不满意。支票和以前一样开始。`'a`替换为占位符`'0`IMPL特征引用被实例化为`Foo<&'static isize>`. 当我们把这两者联系起来时，我们得到了一个约束`'static == '0`. 这意味着污点`'0`是`{'0,
'static}`，泄漏检查失败。

**托多**：这是因为`'static`不是区域变量，但在污染集中，对吗？

## 更高等级的特性义务

一旦基本匹配完成，我们将讨论另一个有趣的主题：如何处理IMPL义务。我将在这里完成一个简单的例子。假设我们有这些特点`Foo`和`Bar`以及相关的IMPL：

```rust
trait Foo<X> {
    fn foo(&self, x: X) { }
}

trait Bar<X> {
    fn bar(&self, x: X) { }
}

impl<X,F> Foo<X> for F
    where F : Bar<X>
{
}
```

现在假设我们有义务`Baz: for<'a> Foo<&'a isize>`我们匹配这个IMPL。因此产生了什么义务？我们想要得到`Baz: for<'a> Bar<&'a isize>`但这是怎么发生的呢？

匹配之后，我们处于这样一个位置，我们有一个占位符替换`X => &'0 isize`. 如果我们将此替换应用于IMPL义务，我们将得到`F : Bar<&'0 isize>`. 显然，这不能直接使用，因为占位符区域`'0`不能泄露我们的计算结果。

我们要做的是从`'0`返回到原始绑定区域（`'a`这里）`'0`导致。（这是在`higher_ranked::plug_leaks`）我们知道泄漏检查通过了，所以这个污点集仅仅由占位符区域本身加上各种中间区域变量组成。然后，我们遍历特征引用，并将该污点集中的每个区域转换回一个后期绑定的区域，因此在本例中，我们将最终得到`Baz: for<'a> Bar<&'a isize>`.
