# 隐含边界

隐式边界消除了在类型声明或特征声明上重复WHERE子句的需要。例如，假设我们有以下类型声明：

```rust,ignore
struct HashSet<K: Hash> {
    ...
}
```

然后我们使用的任何地方`HashSet<K>`作为“输入”类型，出现在`impl`或者在函数的参数中，我们不需要重复`where K: Hash`绑定，如：

```rust,ignore
// I don't want to have to repeat `where K: Hash` here.
impl<K> HashSet<K> {
    ...
}

// Same here.
fn loud_insert<K>(set: &mut HashSet<K>, item: K) {
    println!("inserting!");
    set.insert(item);
}
```

注意在`loud_insert`例子，`HashSet<K>`不是`set`论证`loud_insert`它只是*出现*在参数类型中`&mut HashSet<K>`：我们关心函数头中出现的每个类型（头是没有返回类型的签名），而不仅仅是函数参数的类型。

将隐含边界应用于输入类型的基本原理是，例如，为了调用`loud_insert`上面的函数，程序员必须*产生*类型`HashSet<K>`已经，因此编译器已经验证了`HashSet<K>`形状很好，也就是说`K`有效实施`Hash`，如下例所示：

```rust,ignore
fn main() {
    // I am producing a value of type `HashSet<i32>`.
    // If `i32` was not `Hash`, the compiler would report an error here.
    let set: HashSet<i32> = HashSet::new();
    loud_insert(&mut set, 5);
}
```

因此，我们不想为输入类型重复WHERE子句，因为这会有点重复程序员的工作，在调用函数和在函数参数中使用它们时，必须验证它们的类型的格式是否正确。当使用`impl`.

同样，给出以下特征声明：

```rust,ignore
trait Copy where Self: Clone { // desugared version of `Copy: Clone`
    ...
}
```

然后我们去的每一个地方`SomeType: Copy`我们希望能够利用`SomeType: Clone`不必写得很清楚，如：

```rust,ignore
fn loud_clone<T: Clone>(x: T) {
    println!("cloning!");
    x.clone();
}

fn fun_with_copy<T: Copy>(x: T) {
    println!("will clone a `Copy` type soon...");

    // I'm using `loud_clone<T: Clone>` with `T: Copy`, I know this
    // implies `T: Clone` so I don't want to have to write it explicitly.
    loud_clone(x);
}
```

特征的隐含边界的基本原理是，如果类型实现`Copy`，也就是说，如果存在`impl Copy`对于那种类型，有*应该*存在于`impl Clone`对于该类型，否则编译器会首先报告一个错误。所以，如果我们被迫重复`where SomeType: Clone`但我们已经知道`SomeType: Copy`等等，我们会复制验证工作。

隐含的界限在RustC中还没有完全强制执行，目前它只适用于离群需求、超级特性界限和关联类型的界限。可以找到完整的RFC[在这里][rfc]. 我们将在这里简要介绍隐含边界是如何工作的，以及为什么我们选择以这种方式实现它。在相应的[章](./lowering-rules.md).

[rfc]: https://github.com/rust-lang/rfcs/blob/master/text/2089-implied-bounds.md

## 隐含的界限和降低规则

现在我们需要用逻辑规则表达隐含的界限。我们将从暴露一种幼稚的方式开始。假设我们有以下特点：

```rust,ignore
trait Foo {
    ...
}

trait Bar where Self: Foo { } {
    ...
}
```

所以我们想说如果一个类型实现`Bar`，那么它也必须实现`Foo`. 我们可能认为这样的条款是可行的：

```text
forall<Type> {
    Implemented(Type: Foo) :- Implemented(Type: Bar).
}
```

现在假设我们只编写这个IMPL：

```rust,ignore
struct X;

impl Bar for X { }
```

显然，这是不允许的：事实上，我们写了`Bar`IMPL`X`但是`Bar`特性要求我们也要实现`Foo`对于`X`我们从未做过。就编译器的功能而言，如下所示：

```rust,ignore
struct X;

impl Bar for X {
    // We are in a `Bar` impl for the type `X`.
    // There is a `where Self: Foo` bound on the `Bar` trait declaration.
    // Hence I need to prove that `X` also implements `Foo` for that impl
    // to be legal.
}
```

所以编译器会试图证明`Implemented(X: Foo)`. 当然找不到了`impl Foo for X`因为我们没有写任何东西。然而，它将看到我们隐含的约束条款：

```text
forall<Type> {
    Implemented(Type: Foo) :- Implemented(Type: Bar).
}
```

以便能够证明`Implemented(X: Foo)`如果`Implemented(X: Bar)`持有。结果是`Implemented(X: Bar)`自从我们写了一个`Bar`IMPL`X`！因此编译器将接受`Bar`尽管它不应该这样做。

## 来自环境的隐含界限

所以这种幼稚的方法行不通。我们需要做的是以某种方式将隐含的边界与impl分离。假设我们知道`SomeType<...>`器具`Bar`我们想推断一下`SomeType<...>`还必须实现`Foo`.

有两种可能：第一，我们有足够的信息`SomeType<...>`要知道存在一个`Bar`在包含`SomeType<...>`，例如平原`impl<...> Bar for SomeType<...>`. 如果编译器已经正确地完成了它的工作，那么*必须*存在一`Foo`包括哪些内容`SomeType<...>`例如，另一个平原`impl<...> Foo for SomeType<...>`. 在这种情况下，我们可以使用这个impl，根本不需要隐含的边界。

第二种可能性：我们对`SomeType<...>`为了找到一个`Bar`例如，如果`SomeType<...>`只是函数中的类型参数：

```rust,ignore
fn foo<T: Bar>() {
    // We'd like to deduce `Implemented(T: Foo)`.
}
```

也就是说，信息`T`器具`Bar`这里来自*环境*. 环境是一套我们认为是真的东西，当我们键入检查一些生锈声明。在这种情况下，我们假设`T: Bar`. 那么，在那一点上，我们可以授权自己进行某种“局部的”隐含的约束推理，也就是说`Implemented(T: Foo) :- Implemented(T: Bar)`. 这种推理只能在我们的`foo`函数的作用是为了避免前面有全局子句的问题。

我们可以在任何地方应用这些局部推理，我们都有一个环境——也就是说，当我们可以编写WHERE子句时——也就是说，在IMPLS、特性声明和类型声明中。

## 计算隐含边界`FromEnv`

前一小节表明，只需计算来自环境的事实的隐含界限。我们讨论了“局部”规则，但实际上有多种可能的策略来实现隐含边界的局部性。

在Rustc，当前的策略是*精心制作的*边界：也就是说，每次我们在环境中有一个事实时，我们都递归地派生出这个事实所隐含的所有其他事物，直到我们到达一个固定点为止。例如，如果我们有以下声明：

```rust,ignore
trait A { }
trait B where Self: A { }
trait C where Self: B { }

fn foo<T: C>() {
    ...
}
```

然后在里面`foo`函数，我们从一个只包含`Implemented(T: C)`. 那么因为暗含的界限`C`特性，我们详细说明`Implemented(T: B)`把它添加到我们的环境中。因为`B`特性，我们详细说明`Implemented(T: A)`并将其添加到我们的环境中。我们不能阐述任何其他的事情，因此我们得出结论，我们的最终环境包括`Implemented(T: A + B + C)`.

在新的特征系统中，我们喜欢用逻辑规则尽可能多地编码。因此，我们有一套*全球的*程序条款定义如下：

```text
forall<T> { Implemented(T: A) :- FromEnv(T: A). }

forall<T> { Implemented(T: B) :- FromEnv(T: B). }
forall<T> { FromEnv(T: A) :- FromEnv(T: B). }

forall<T> { Implemented(T: C) :- FromEnv(T: C). }
forall<T> { FromEnv(T: C) :- FromEnv(T: C). }
```

因此，这些子句是全局定义的（也就是说，它们在程序中的任何地方都可用），但它们不能被使用，因为假设的形式总是`FromEnv(...)`有点特别。的确，正如名字所示，`FromEnv(...)`事实可以**只有**来自环境。它的工作原理是`foo`函数，而不是具有包含`Implemented(T: C)`，我们将此环境替换为`FromEnv(T: C)`.从这里开始，由于上述条款，我们看到我们能够达到`Implemented(T: A)`，请`Implemented(T: B)`或`Implemented(T: C)`这就是我们想要的。

## 隐含边界和格式良好性检查

隐含边界与形式良好性检查密切相关。格式良好检查是检查程序员编写的impl是否合法的过程，我们之前称之为“编译器正确地执行其工作”。

我们已经看到了非法和合法实施的例子：

```rust,ignore
trait Foo { }
trait Bar where Self: Foo { }

struct X;
struct Y;

impl Bar for X {
    // This impl is not legal: the `Bar` trait requires that we also
    // implement `Foo`, and we didn't.
}

impl Foo for Y {
    // This impl is legal: there is nothing to check as there are no where
    // clauses on the `Foo` trait.
}

impl Bar for Y {
    // This impl is legal: we have a `Foo` impl for `Y`.
}
```

我们必须明确“合法”和“非法”的含义。为此，我们引入另一个谓词：`WellFormed(Type: Trait)`.我们说性状参考`Type: Trait`如果`Type`符合上写的界限`Trait`宣言。对于我们编写的每个IMPL，假设在IMPL上声明的WHERE子句保持不变，编译器将尝试证明相应的特征引用是格式良好的。如果编译器成功地做到了这一点，那么IMPL是合法的。

开始定义`WellFormed(Type: Trait)`将其定义为：

```rust,ignore
trait Trait where WC1, WC2, ..., WCn {
    ...
}
```

```text
forall<Type> {
    WellFormed(Type: Trait) :- WC1 && WC2 && .. && WCn.
}
```

实际上，这基本上就是在Rustc中所做的，直到人们注意到它与隐含的界限混在一起。关键是，隐含边界允许某人导出环境中某个事实所隐含的所有边界，并且*及物的*正如我们看到的`A + B + C`特性示例。然而，`WellFormed`上面定义的谓词只检查*直接的*超边界保持。也就是说，如果我们回到`A + B + C`例子：

```rust,ignore
trait A { }
// No where clauses, always well-formed.
// forall<Type> { WellFormed(Type: A). }

trait B where Self: A { }
// We only check the direct superbound `Self: A`.
// forall<Type> { WellFormed(Type: B) :- Implemented(Type: A). }

trait C where Self: B { }
// We only check the direct superbound `Self: B`. We do not check
// the `Self: A` implied bound  coming from the `Self: B` superbound.
// forall<Type> { WellFormed(Type: C) :- Implemented(Type: B). }
```

隐含界的递归幂与`WellFormed`. 事实证明，这种不对称可以[剥削][bug]. 事实上，假设我们定义了以下特征：

```rust,ignore
trait Partial where Self: Copy { }
// WellFormed(Self: Partial) :- Implemented(Self: Copy).

trait Complete where Self: Partial { }
// WellFormed(Self: Complete) :- Implemented(Self: Partial).

impl<T> Partial for T where T: Complete { }

impl<T> Complete for T { }
```

对于`Partial`IMPL，编译器必须证明的是：

```text
forall<T> {
    if (T: Complete) { // assume that the where clauses hold
        WellFormed(T: Partial) // show that the trait reference is well-formed
    }
}
```

证明`WellFormed(T: Partial)`相当于证明`Implemented(T: Copy)`. 但是，我们有`Implemented(T: Complete)`在我们的环境中：由于隐含的界限，我们可以推断`Implemented(T: Partial)`. 使用一个层次的隐含边界，我们可以推断`Implemented(T: Copy)`. 最后，`Partial`IMPL是合法的。

对于`Complete`IMPL，编译器必须证明的是：

```text
forall<T> {
    WellFormed(T: Complete) // show that the trait reference is well-formed
}
```

证明`WellFormed(T: Complete)`相当于证明`Implemented(T: Partial)`. 我们看到了`impl Partial for T`如果我们能证明`Implemented(T: Complete)`事实证明我们可以证明这个事实`impl<T> Complete for T`是一个没有任何where子句的总括impl。

所以这两个impl都是合法的，编译器接受程序。而且，多亏了`Complete`所有类型的实现`Complete`. 所以我们现在可以像这样使用这个IMPL：

```rust,ignore
fn eat<T>(x: T) { }

fn copy_everything<T: Complete>(x: T) {
    eat(x);
    eat(x);
}

fn main() {
    let not_copiable = vec![1, 2, 3, 4];
    copy_everything(not_copiable);
}
```

在这个程序中，我们使用的事实是`Vec<i32>`器具`Complete`和其他类型一样。所以我们可以打电话`copy_everything`带有类型的参数`Vec<i32>`. 里面`copy_everything`函数，我们有`Implemented(T: Complete)`在我们的环境中。由于隐含的界限，我们可以推断`Implemented(T: Partial)`.再利用隐含边界，我们推导出`Implemented(T: Copy)`我们真的可以称之为`eat`函数，它将参数移动两次，因为它的参数是`Copy`.问题：The`T`实际上类型是`Vec<i32>`这根本不是复制的，因此我们将双倍释放底层的vec存储，这样我们就可以在安全的信任中拥有不健全的内存。

当然，忽略了`WellFormed`以及隐含的边界，这个bug是可能的，因为我们有某种自引用的impl。但自我参照的暗示在实践中非常有用，并不是这件事的真正罪魁祸首。

[bug]: https://github.com/rust-lang/rust/pull/43786

## 共同诱导性`WellFormed`

所以解决方法是解决`WellFormed`以及隐含的界限。为此，我们需要`WellFormed`谓词不仅要求直接上界保持不变，而且还要求上界传递地隐含的所有边界。我们可以做的是为`WellFormed`谓语：

```rust,ignore
trait A { }
// WellFormed(Self: A) :- Implemented(Self: A).

trait B where Self: A { }
// WellFormed(Self: B) :- Implemented(Self: B) && WellFormed(Self: A).

trait C where Self: B { }
// WellFormed(Self: C) :- Implemented(Self: C) && WellFormed(Self: B).
```

注意，我们现在还要求`Implemented(Self: Trait)`对于`WellFormed(Self: Trait)`为真：这是为了简化以传递方式遍历所有隐含边界的过程。当检查impl是否合法时，这不会改变任何东西，因为我们假设在impl中包含的where子句，我们知道相应的特征引用确实有效。由于这个设置，您可以看到我们确实需要证明由WHERE子句传递地隐含的所有边界集。

然而，仍然有一个陷阱。假设我们有以下特征定义：

```rust,ignore
trait Foo where <Self as Foo>::Item: Foo {
    type Item;
}
```

所以这个定义比我们已经看到的定义要复杂一些，因为它定义了一个关联项。然而，形式良好的规则不会更复杂：

```text
WellFormed(Self: Foo) :-
    Implemented(Self: Foo) &&
    WellFormed(<Self as Foo>::Item: Foo).
```

现在，我们要编写以下IMPL：

```rust,ignore
impl Foo for i32 {
    type Item = i32;
}
```

这个`Foo`特征定义和`impl Foo for i32`是完全有效的rust：我们有点递归地使用`Foo`impl以显示关联值确实实现`Foo`但没关系。但是，如果我们将其转换为我们的格式良好的设置，则编译器内部的证明过程`Foo`IMPL是这样的：它从证明良好形式的目标开始`WellFormed(i32: Foo)`是真的。为此，必须证明以下目标：`Implemented(i32: Foo)`和`WellFormed(<i32 as Foo>::Item: Foo)`. `Implemented(i32: Foo)`之所以成立，是因为它有IMPL，并且没有WHERE子句，所以它总是正确的。但是，由于我们使用的关联类型值，`WellFormed(<i32 as Foo>::Item: Foo)`简化为`WellFormed(i32: Foo)`.所以为了证明它最初的目标`WellFormed(i32: Foo)`，编译器需要证明`WellFormed(i32: Foo)`：这显然是一个周期，周期通常被特征解算器拒绝，除非…如果`WellFormed`谓词是共归纳的。

共归纳谓词，如第[目标和条款](./goals-and-clauses.md#coinductive-goals)，是特征解算器接受循环的谓词。在我们的环境中，这将是一件有效的事情：事实上，`WellFormed`谓词只是一种枚举所有隐含边界的方法。因此，它就像一个不动点算法：它试图增加一组隐含的边界，直到没有更多的要添加的内容为止。在这里，一个循环在`WellFormed`谓词只是意味着在这个方向上没有更多的界限可以添加，所以我们可以接受这个循环，并关注其他方向。很容易证明，在这些共归纳语义下，我们有效地访问了所有可传递的隐含界限，并且仅访问了这些。

## 类型的隐含边界

我们主要讨论特性的隐含界限，因为这是实现方面最微妙的。类型上的隐含边界更简单，特别是因为如果我们假设一个类型是格式良好的，我们不使用这个事实来推断其他类型是格式良好的，我们只使用它来推断，例如某些特性边界保持不变。

对于类型，我们只使用如下规则：

```rust,ignore
struct Type<...> where WC1, ..., WCn {
    ...
}
```

```text
forall<...> {
    WellFormed(Type<...>) :- WC1, ..., WCn.
}

forall<...> {
    FromEnv(WC1) :- FromEnv(Type<...>).
    ...
    FromEnv(WCn) :- FromEnv(Type<...>).
}
```

我们可以看到，在形式良好性检查（仅验证直接超界限是否存在）和隐含界限（允许访问由where子句传递性隐含的所有界限）之间存在这种不对称性。在这种情况下，这是可以的，因为正如我们所说，我们不使用`FromEnv(Type<...>)`演绎其他`FromEnv(OtherType<...>)`东西，我们也不用`FromEnv(Type: Trait)`推论`FromEnv(OtherType<...>)`东西。所以从这个意义上说，类型定义比特征“不那么递归”，我们在前面的小节中看到，是不对称和递归特征/隐含的结合导致了不健全。这样，`WellFormed(Type<...>)`谓词不需要是共归纳的。

这种不对称优化是有用的，因为在真正的Rust程序中，我们必须经常检查类型的良好形式（例如，对于出现在函数体中的每个类型）。
