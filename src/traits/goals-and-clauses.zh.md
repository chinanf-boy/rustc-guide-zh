# 目标和条款

在逻辑编程术语中，a**目标**是你必须证明的东西**条款**你知道的是真的。如中所述[降低到逻辑](./lowering-to-logic.html)第二章，Rust的特征解算是基于遗传harrop（hh）子句的扩展，它用几个新的超级大国扩展了传统的prolog-horn子句。

## 目标和条款元结构

在Rust的解决方案中，**目标**和**条款**具有以下形式（请注意，这两个定义相互引用）：

```text
Goal = DomainGoal           // defined in the section below
        | Goal && Goal
        | Goal || Goal
        | exists<K> { Goal }   // existential quantification
        | forall<K> { Goal }   // universal quantification
        | if (Clause) { Goal } // implication
        | true                 // something that's trivially true
        | ambiguous            // something that's never provable

Clause = DomainGoal
        | Clause :- Goal     // if can prove Goal, then Clause is true
        | Clause && Clause
        | forall<K> { Clause }

K = <type>     // a "kind"
    | <lifetime>
```

这些目标的证明过程实际上非常简单。本质上，这是深度优先搜索的一种形式。论文[“遗传哈罗普公式逻辑的证明程序”][pphhf]提供详细信息。

在代码方面，这些类型在[`librustc/traits/mod.rs`][traits_mod]在Rustc和[`chalk-ir/src/lib.rs`][chalk_ir]用粉笔画。

[pphhf]: ./bibliography.html#pphhf

[traits_mod]: https://github.com/rust-lang/rust/blob/master/src/librustc/traits/mod.rs

[chalk_ir]: https://github.com/rust-lang-nursery/chalk/blob/master/chalk-ir/src/lib.rs

<a name="domain-goals"></a>

## 领域目标

*领域目标*是特性逻辑的原子。正如在上面给出的定义中所看到的，一般目标基本上由域目标组合而成。

此外，将前面给出的从句定义展平一点，我们可以看到从句的形式总是：

```text
forall<K1, ..., Kn> { DomainGoal :- Goal }
```

因此，领域目标实际上是条款的lhs。也就是说，在最细微的层面上，领域目标是特性解决者最终想要证明的。

<a name="trait-ref"></a>

为了在我们的系统中定义一组领域目标，我们需要首先介绍一些简单的公式。一**性状参考**由一个特性的名称和一组合适的输入p0..pn组成：

```text
TraitRef = P0: TraitName<P1..Pn>
```

例如，`u32: Display`是一个特征参照物`Vec<T>:
IntoIterator`.请注意，rust surface语法还允许一些额外的东西，如关联的类型绑定（`Vec<T>: IntoIterator<Item =
T>`不属于特征参照的一部分。

<a name="projection"></a>

A **投影**包含关联项引用及其输入p0..pm：

```text
Projection = <P0 as TraitName<P1..Pn>>::AssocItem<Pn+1..Pm>
```

考虑到这些，我们可以定义`DomainGoal`如下：

```text
DomainGoal = Holds(WhereClause)
            | FromEnv(TraitRef)
            | FromEnv(Type)
            | WellFormed(TraitRef)
            | WellFormed(Type)
            | Normalize(Projection -> Type)

WhereClause = Implemented(TraitRef)
            | ProjectionEq(Projection = Type)
            | Outlives(Type: Region)
            | Outlives(Region: Region)
```

`WhereClause`指的是`where`Rust用户实际上能够在Rust程序中编写的条款。这种抽象的存在只是为了方便起见，因为我们有时只想处理在rust中有效可写的域目标。

让我们逐一分解每一个。

#### 已实施（traitref）

例如`Implemented(i32: Copy)`

如果给定的特性是为给定的输入类型和生存期实现的，则为true。

#### ProjectIOneQ（投影=类型）

例如`ProjectionEq<T as Iterator>::Item = u8`

给定的关联类型`Projection`等于`Type`；这可以通过规范化或使用与占位符关联的类型来证明。参见[有关关联类型的部分](./associated-types.html).

#### 规格化（投影->类型）

例如`ProjectionEq<T as Iterator>::Item -> u8`

给定的关联类型`Projection`可以是[归一化][n]到`Type`.

正如在[有关关联类型的部分](./associated-types.html)，`Normalize`暗示`ProjectionEq`反之亦然。一般来说，证明`Normalize(<T as Trait>::Item -> U)`也需要证明`Implemented(T: Trait)`.

[n]: ./associated-types.html#normalize

[at]: ./associated-types.html

#### 来自env（traitref）

例如`FromEnv(Self: Add<i32>)`

如果内部`TraitRef`是*假定的*也就是说，如果它可以从in-scope-where子句派生出来。

例如，给定以下函数：

```rust
fn loud_clone<T: Clone>(stuff: &T) -> T {
    println!("cloning!");
    stuff.clone()
}
```

在我们的身体里，我们会`FromEnv(T: Clone)`. 在范围WHERE子句嵌套中，因此IMPL主体内的函数体也继承IMPL主体的WHERE子句。

此规则和下一个规则用于实现[隐含边界]. 正如我们在下降部分看到的，`FromEnv(TraitRef)`暗示`Implemented(TraitRef)`反之亦然。这种区别对于隐含的界限至关重要。

#### FromEnv（类型）

例如`FromEnv(HashSet<K>)`

如果内部`Type`是*假设*如果它是一个函数或IMPL的输入类型，那么它的格式就很好。

例如，给定以下代码：

```rust,ignore
struct HashSet<K> where K: Hash { ... }

fn loud_insert<K>(set: &mut HashSet<K>, item: K) {
    println!("inserting!");
    set.insert(item);
}
```

`HashSet<K>`是的输入类型`loud_insert`功能。因此，我们假设它是成形良好的，所以我们应该`FromEnv(HashSet<K>)`在我们的身体里。正如我们在下降部分看到的，`FromEnv(HashSet<K>)`暗示`Implemented(K: Hash)`因为`HashSet`声明是用`K: Hash`WHERE子句。因此，我们不需要在`loud_insert`函数：我们宁愿自动假定它是真的。

#### 成形良好（项目）

这些目标意味着给定的项目是*成形良好*.

我们可以讨论不同类型的项目形成良好：

-   *类型*，就像`WellFormed(Vec<i32>)`在铁锈中是正确的，或`WellFormed(Vec<str>)`，这不是（因为`str`不是`Sized`。）

-   *卖国贼*，就像`WellFormed(Vec<i32>: Clone)`.

井然有序对[隐含边界]. 特别是可以假设的原因`FromEnv(T: Clone)`在`loud_clone`例如，我们*也*验证`WellFormed(T: Clone)`对于每个呼叫站点`loud_clone`. 同样，可以假设`FromEnv(HashSet<K>)`在`loud_insert`示例，因为我们将验证`WellFormed(HashSet<K>)`对于每个呼叫站点`loud_insert`. 

#### 异常值（类型：区域）、异常值（区域：区域）

例如`Outlives(&'a str: 'b)`，`Outlives('a: 'static)`

如果左边给定的类型或区域超过右边区域，则为true。

<a name="coinductive"></a>

## 共同目标

我们系统中的大多数目标都是“归纳的”。在归纳目标中，不允许循环推理。考虑这个示例子句：

```text
    Implemented(Foo: Bar) :-
        Implemented(Foo: Bar).
```

如果我们试图证明`Implemented(Foo: Bar)`，然后递归地证明`Implemented(Foo: Bar)`循环将无限地继续（特征解算器将在这里终止，它只考虑`Implemented(Foo: Bar)`不知道是真的）。

但是，有些目标是*共感性*. 简而言之，这意味着循环是可以的。所以，如果`Bar`如果是一个共感性状，那么上面的规则是完全有效的，它将表明`Implemented(Foo: Bar)`是真的。

*自动特征*是使用共感应目标的铁锈的一个例子。考虑一下`Send`特征，想象我们有这个结构：

```rust
struct Foo {
    next: Option<Box<Foo>>
}
```

自动特征的默认规则是这样说的`Foo`是`Send`如果其字段的类型为`Send`. 因此，我们有一个规则

```text
Implemented(Foo: Send) :-
    Implemented(Option<Box<Foo>>: Send).
```

正如你可能想象的那样，证明`Option<Box<Foo>>: Send`最终会以循环的方式要求我们证明`Foo: Send`再一次。所以这是一个例子，我们以一个周期结束，但没关系，我们*做*考虑`Foo: Send`保留，即使它引用自己。

一般来说，当我们想列举一组固定的可能性时，共同归纳特征被用于解决锈病。在自动特征的情况下，我们从给定的起点（即`Foo`可以达到类型的值`Option<Box<Foo>>`，这意味着它可以达到类型的值`Box<Foo>`，然后是类型`Foo`，然后循环完成）。

除了自我特征外，`WellFormed`谓词是共归纳的。它们用于实现类似的“枚举所有案例”模式，如[隐含边界].

[implied bounds]: ./lowering-rules.html#implied-bounds

## 不完整章节

还有一些主题有待编写：

-   详细说明证明程序
-   SLG解决方案–引入负面推理
