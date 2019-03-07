# 下沉规则

本节给出了铁锈特性的完整降低规则[纲领性条款][pc]. 这是一种参考。这些规则引用了[领域目标][dg]在前面的部分中定义。

[pc]: ./goals-and-clauses.html

[dg]: ./goals-and-clauses.html#domain-goals

## 记数法

非终结符`Pi`是指一些一般的*参数*或者是一个命名的一生`'a`或类似的类型参数`A`.

非终结符`Ai`是指一些一般的*论点*可能是一辈子`'a`或类似类型的`Vec<A>`.

在定义降低规则时，我们将在[本节给出的符号](./goals-and-clauses.html). 我们有时会像这样插入“宏”`LowerWhereClause!`在这些定义中，这些宏引用了本章中的其他部分。

## 规则名称和交叉引用

这些降低规则中的每一个都有一个名称，并记录了如下注释：

```
// Rule Foo-Bar-Baz
```

这些规则的参考实施见[`chalk/src/rules.rs`][chalk_rules]. 它们也被移植到[`librustc_traits`][librustc_traits]机箱。

[chalk_rules]: https://github.com/rust-lang-nursery/chalk/blob/master/src/rules.rs

[librustc_traits]: https://github.com/rust-lang/rust/tree/master/src/librustc_traits

## 降低WHERE子句

当用于目标位置时，where子句可以直接映射到`Holds`变异体[领域目标][dg]如下：

-   `A0: Foo<A1..An>`地图到`Implemented(A0: Foo<A1..An>)`
-   `T: 'r`地图到`Outlives(T, 'r)`
-   `'a: 'b`地图到`Outlives('a, 'b)`
-   `A0: Foo<A1..An, Item = T>`有点特别，可以扩展到两个不同的目标，即`Implemented(A0: Foo<A1..An>)`和`ProjectionEq(<A0 as Foo<A1..An>>::Item = T)`

在下面的规则中，我们将使用`WC`指明rust语法中出现的子句的位置；然后我们将使用相同的`WC`在我们正在生成的程序子句中，指出那些where子句作为目标出现在哪里。在这种情况下，上面的映射用于将rust语法转换为目标。

### 转换降低的where子句

此外，在下面的规则中，我们有时会对下面定义的where子句进行一些转换：

-   `FromEnv(WC)`–这表明：
    -   `Implemented(TraitRef)`变成`FromEnv(TraitRef)`
    -   其他条款完整无缺
-   `WellFormed(WC)`–这表明：
    -   `Implemented(TraitRef)`变成`WellFormed(TraitRef)`
    -   其他条款完整无缺

*托多*：我怀疑我们也想改变异常值关系，但乔克现在没有建模。

## 降低性状

给出特征定义

```rust,ignore
trait Trait<P1..Pn> // P0 == Self
where WC
{
    // trait items
}
```

我们将生成一些声明。本节主要讨论特性头的程序子句（即`{}`；[特色项目部分](#trait-items)把里面的东西盖住`{}`.

### 特性标题

从特性本身来说，我们主要制定“元”规则来建立不同领域目标之间的关系。特性头中的第一个这样的规则创建`FromEnv`和`Implemented`谓词：

```text
// Rule Implemented-From-Env
forall<Self, P1..Pn> {
  Implemented(Self: Trait<P1..Pn>) :- FromEnv(Self: Trait<P1..Pn>)
}
```

<a name="implied-bounds"></a>

#### 隐含界限

接下来的几个子句与隐含的界限有关（另请参见[RFC 2089]以及[隐含界限][implied_bounds]更深入的介绍）。对于每个特征，我们产生两个从句：

[rfc 2089]: https://rust-lang.github.io/rfcs/2089-implied-bounds.html

[implied_bounds]: ./implied-bounds.md

```text
// Rule Implied-Bound-From-Trait
//
// For each where clause WC:
forall<Self, P1..Pn> {
  FromEnv(WC) :- FromEnv(Self: Trait<P1..Pn)
}
```

这个子句说，如果我们假设特性成立，那么我们也可以假设它的where子句成立。看到这样一个例子也许很有用：

```rust,ignore
trait Eq: PartialEq { ... }
```

在这种情况下，`PartialEq`超特性等价于`where
Self: PartialEq`在我们的简化模型中。因此，上述程序条款规定，如果我们能够证明`FromEnv(T: Eq)`–例如，如果我们在`T: Eq`在WHERE条款中，我们也知道`FromEnv(T: PartialEq)`.因此，从环境中产生的一系列事物不仅是**直接WHERE子句**但也会从中得到一些东西。

下一条规则是相关的；它定义了特征引用的含义**成形良好**以下内容：

```text
// Rule WellFormed-TraitRef
forall<Self, P1..Pn> {
  WellFormed(Self: Trait<P1..Pn>) :- Implemented(Self: Trait<P1..Pn>) && WellFormed(WC)
}
```

这个`WellFormed`规则规定`T: Trait`如果（a）形成良好`T: Trait`执行和（b）在`Trait`形成良好（因此得以实施）。记住`WellFormed`谓词是[共同生产的](./goals-and-clauses.html#coinductive)；在这种情况下，它是一种“载体”，允许我们列举所有通过`T: Trait`.

一个例子：

```rust,ignore
trait Foo: A + Bar { }
trait Bar: B + Foo { }
trait A { }
trait B { }
```

这里，传递的一组含义`T: Foo`是`T: A`，请`T: Bar`，和`T: B`.事实上，如果我们试图证明`WellFormed(T: Foo)`，我们必须证明其中的每一个：

-   `WellFormed(T: Foo)`
    -   `Implemented(T: Foo)`
    -   `WellFormed(T: A)`
        -   `Implemented(T: A)`
    -   `WellFormed(T: Bar)`
        -   `Implemented(T: Bar)`
        -   `WellFormed(T: B)`
            -   `Implemented(T: Bar)`
        -   `WellFormed(T: Foo)`--循环，真正的共同生产

这个`WellFormed`谓词仅在证明impl格式正确时使用，基本上，对于某些特征引用的每个impl`TraitRef`我们必须证明`WellFormed(TraitRef)`.这反过来证明了隐含的边界规则允许我们扩展`FromEnv`项目。

## 降低类型定义

我们还希望有一些规则来定义类型是什么时候形成的。例如，给定此类型：

```rust,ignore
struct Set<K> where K: Hash { ... }
```

然后`Set<i32>`形成良好，因为`i32`工具`Hash`，但是`Set<NotHash>`不会形成良好的形状。基本上，如果类型的参数验证了写在类型定义上的WHERE子句，则该类型的格式是良好的。

因此，对于每个类型定义：

```rust, ignore
struct Type<P1..Pn> where WC { ... }
```

我们得出以下规则：

```text
// Rule WellFormed-Type
forall<P1..Pn> {
  WellFormed(Type<P1..Pn>) :- WC
}
```

注意我们使用`struct`用于定义类型，但应将其理解为常规类型定义（它可以是泛型`enum`）。

相反，我们定义规则，如果我们假设一个类型是格式良好的，我们也可以假设它的WHERE子句持有。也就是说，我们制定了以下规则系列：

```text
// Rule Implied-Bound-From-Type
//
// For each where clause `WC`
forall<P1..Pn> {
  FromEnv(WC) :- FromEnv(Type<P1..Pn>)
}
```

对于隐含的边界RFC，函数将*假定*他们的论点很有条理。例如，假设我们有以下代码位：

```rust,ignore
trait Hash: Eq { }
struct Set<K: Hash> { ... }

fn foo<K>(collection: Set<K>, x: K, y: K) {
    // `x` and `y` can be equalized even if we did not explicitly write
    // `where K: Eq`
    if x == y {
        ...
    }
}
```

在`foo`函数，我们假设`Set<K>`结构良好，即我们`FromEnv(Set<K>)`在我们的环境中。因为前面的规则，我们`FromEnv(K: Hash)`不需要显式的WHERE子句。因为`Hash`特征定义，也有一个规则说：

```text
forall<K> {
  FromEnv(K: Eq) :- FromEnv(K: Hash)
}
```

这意味着我们最终`FromEnv(K: Eq)`然后可以比较`x`和`y`不需要显式的WHERE子句。

<a name="trait-items"></a>

## 降低特征项

### 关联的类型声明

给定一个声明（可能是泛型）关联类型的特性：

```rust,ignore
trait Trait<P1..Pn> // P0 == Self
where WC
{
    type AssocType<Pn+1..Pm>: Bounds where WC1;
}
```

我们将生成许多程序子句。前两个定义规则`ProjectionEq`可以成功；这两个条款在[有关关联类型的部分](./associated-types.html)，但此处转载供参考：

```text
// Rule ProjectionEq-Normalize
//
// ProjectionEq can succeed by normalizing:
forall<Self, P1..Pn, Pn+1..Pm, U> {
  ProjectionEq(<Self as Trait<P1..Pn>>::AssocType<Pn+1..Pm> = U) :-
      Normalize(<Self as Trait<P1..Pn>>::AssocType<Pn+1..Pm> -> U)
}
```

```text
// Rule ProjectionEq-Placeholder
//
// ProjectionEq can succeed through the placeholder associated type,
// see "associated type" chapter for more:
forall<Self, P1..Pn, Pn+1..Pm> {
  ProjectionEq(
    <Self as Trait<P1..Pn>>::AssocType<Pn+1..Pm> =
    (Trait::AssocType)<Self, P1..Pn, Pn+1..Pm>
  )
}
```

下一条规则涵盖了投影的隐含边界。尤其是，`Bounds`在关联类型上声明的必须已被证明为保持，以显示IMPL的格式良好，因此我们可以在其他地方依赖它们。

```text
// Rule Implied-Bound-From-AssocTy
//
// For each `Bound` in `Bounds`:
forall<Self, P1..Pn, Pn+1..Pm> {
    FromEnv(<Self as Trait<P1..Pn>>::AssocType<Pn+1..Pm>>: Bound) :-
      FromEnv(Self: Trait<P1..Pn>) && WC1
}
```

接下来，我们定义了将关联类型实例化为格式良好的需求…

```text
// Rule WellFormed-AssocTy
forall<Self, P1..Pn, Pn+1..Pm> {
    WellFormed((Trait::AssocType)<Self, P1..Pn, Pn+1..Pm>) :-
      Implemented(Self: Trait<P1..Pn>) && WC1
}
```

…以及相反的含义，当我们可以假设它是形成良好的。

```text
// Rule Implied-WC-From-AssocTy
//
// For each where clause WC1:
forall<Self, P1..Pn, Pn+1..Pm> {
    FromEnv(WC1) :- FromEnv((Trait::AssocType)<Self, P1..Pn, Pn+1..Pm>)
}
```

```text
// Rule Implied-Trait-From-AssocTy
forall<Self, P1..Pn, Pn+1..Pm> {
    FromEnv(Self: Trait<P1..Pn>) :-
      FromEnv((Trait::AssocType)<Self, P1..Pn, Pn+1..Pm>)
}
```

### 降低函数和常量声明

粉笔没有建立函数和常量的模型，但我最终希望把它们当作规范化处理。见[下面的函数/常量值部分](#constant-vals)了解更多详细信息。

## 下沉暗示

给了一个特征的暗示：

```rust,ignore
impl<P0..Pn> Trait<A1..An> for A0
where WC
{
    // zero or more impl items
}
```

让`TraitRef`做品质参考`A0: Trait<A1..An>`. 然后我们将创建以下规则：

```text
// Rule Implemented-From-Impl
forall<P0..Pn> {
  Implemented(TraitRef) :- WC
}
```

此外，我们将降低所有*IMPL项目*.

## 降低IMPL项目

### 关联的类型值

给定一个包含以下内容的IMPL：

```rust,ignore
impl<P0..Pn> Trait<P1..Pn> for P0
where WC_impl
{
    type AssocType<Pn+1..Pm> = T;
}
```

以及我们的WHERE条款`WC1`从上面的特征关联类型，我们得出以下规则：

```text
// Rule Normalize-From-Impl
forall<P0..Pm> {
  forall<Pn+1..Pm> {
    Normalize(<P0 as Trait<P1..Pn>>::AssocType<Pn+1..Pm> -> T) :-
      Implemented(P0 as Trait) && WC1
  }
}
```

注意`WC_impl`和`WC1`两者都对IMPL可以依赖的WHERE子句进行编码。（`WC_impl`此处不使用，因为它是由`Implemented(P0 as Trait)`）

<a name="constant-vals"></a>

### 函数和常量值

粉笔没有建立函数和常量的模型，但我最终希望把它们当作规范化处理。这可能需要添加一种新的参数（常量），然后`NormalizeValue`领域目标。这是*被书写*因为细节有点悬而未决。
