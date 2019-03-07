# 区域推理（NLL）

基于MIR的区域检查代码位于[该`rustc_mir::borrow_check::nll`模][nll]。（当然，NLL代表“非词汇生命周期”，这个术语一旦成为标准的生命周期就有望被弃用。）

[nll]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/index.html

基于MIR的区域分析包括两个主要功能：

-   [`replace_regions_in_mir`]，首先调用，有两个工作：
    -   首先，它找到出现在函数签名内的区域集（例如，`'a`在`fn foo<'a>(&'a u32) {
        ... }`）。这些被称为“通用”或“自由”区域 - 特别是，它们是那些区域[显得自由][fvb]在函数体中。
    -   其次，它用新的推理变量替换函数体中的所有区域。这是因为（目前）这些区域是词汇区域推断的结果，因此没有太大兴趣。目的是 - 最终 - 它们将是“擦除区域”（即根本没有信息），因为我们根本不会进行词汇区域推断。
-   [`compute_regions`]，调用第二：这是作为参数给出移动分析的结果。它具有计算所有推理变量的值的工作`replace_regions_in_mir`介绍。
    -   要做到这一点，它首先运行[MIR型检查器]。这基本上是一种普通的类型检查器，但专门用于MIR，当然，它比完整的Rust简单得多。然而，运行MIR类型检查器将创建**超出限制**在区域变量之间（例如，一个变量必须比另一个变量更长）以反映出现的子类型关系。
    -   它还补充说**活力限制**由变量使用的地方引起的。
    -   在此之后，我们创建了一个[`RegionInferenceContext`]使用我们计算的约束和我们引入的推理变量并使用[`solve`]推断所有区域推断变量的值的方法。
    -   该[NLL RFC]还包括相当彻底（并且希望可读）的报道。

[fvb]: ../appendix/background.html#free-vs-bound

[`replace_regions_in_mir`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/fn.replace_regions_in_mir.html

[`compute_regions`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/fn.compute_regions.html

[`regioninferencecontext`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html

[`solve`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#method.solve

[nll rfc]: http://rust-lang.github.io/rfcs/2094-nll.html

[mir type checker]: ./type_check.md

## 环球地区

该[`UnversalRegions`]type表示集合*普遍*区域对应一些MIR`DefId`。它建于[`replace_regions_in_mir`]当我们用新的推理变量替换所有区域时。[`UniversalRegions`]包含给定MIR中所有自由区域的索引以及任何关系*已知*在它们之间保持（例如隐含的界限，条款等）。

例如，给定以下函数的MIR：

```rust
fn foo<'a>(x: &'a u32) {
    // ...
}
```

我们将创建一个通用区域`'a`和一个`'static`。处理闭包可能还有一些复杂因素，但我们暂时会忽略它们。

TODO：写一下*怎么样*计算这些区域。

[`universalregions`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/universal_regions/struct.UniversalRegions.html

## 区域变量

区域的价值可以被认为是**组**。该集合包含MIR中该区域有效的所有点以及该区域失效的任何区域（例如，如果`'a: 'b`， 然后`end('b)`是为了设置`'a`）;我们称之为此集合的域名`RegionElement`。在代码中，所有区域的值都保持在[该`rustc_mir::borrow_check::nll::region_infer`模][ri]。对于每个区域，我们维护一个集合，存储其值中存在的元素（为了使这个有效，我们给每种元素一个索引，`RegionElementIndex`，并使用稀疏位集）。

[ri]: https://github.com/rust-lang/rust/tree/master/src/librustc_mir/borrow_check/nll/region_infer/

区域元素的种类如下：

-   每**地点**在MIR控制流图中：位置只是基本块和索引的对。这标志着这一点**入境时**到具有该索引的语句（或终止符，如果索引等于`statements.len()`）。
-   有一个元素`end('a)`对于每个普遍区域`'a`，对应于呼叫者（或呼叫者的呼叫者等）控制流图的某些部分。
-   类似地，有一个元素表示`end('static)`对应于此函数返回后的其余程序执行。
-   有一个元素`!1`对于每个占位符区域`!1`。这（直观地）对应于一些未知的其他元素 - 有关占位符的详细信息，请参阅本节[占位符和宇宙](#placeholder)。

## 约束

在我们推断区域的价值之前，我们需要收集区域的约束。有两种主要类型的约束。

1.  超出限制。这些是一个区域比另一个区域更长的约束（例如`'a: 'b`）。Outlives约束由生成[MIR型检查器]。
2.  生活限制。每个地区都需要在可以使用的地方居住。这些约束由收集[`generate_constraints`]。

[`generate_constraints`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/constraint_generation/fn.generate_constraints.html

## 推理概述

那么我们如何计算一个地区的内容呢？这个过程被称为*区域推断*。高层次的想法非常简单，但我们需要注意一些细节。

这是一个高层次的想法：我们从每个区域开始，我们知道MIR位置必须来自活跃性约束。从那里，我们使用从类型检查器计算的所有outlives约束*传播*约束：对每个地区`'a`如果`'a: 'b`，然后我们添加所有元素`'b`至`'a`， 包含`end('b)`。这一切都发生在[`propagate_constraints`]。

然后，我们将检查错误。我们首先通过调用检查是否满足类型测试[`check_type_tests`]。这会检查约束`T: 'a`。其次，我们检查通用区域是不是“太大”。这是通过调用完成的[`check_universal_regions`]。这将检查每个区域`'a`如果`'a`包含元素`end('b)`那么我们必须已经知道了`'a: 'b`持有（例如来自where子句）。如果我们还不知道这一点，那就错了......好吧，差不多。闭包有一些特殊处理，我们稍后会讨论。

### 例

请考虑以下示例：

```rust,ignore
fn foo<'a, 'b>(x: &'a usize) -> &'b usize {
    x
}
```

显然，这不应该编译，因为我们不知道是否`'a`会超越`'b`（如果没有，则返回值可能是悬空参考）。

让我们稍微回顾一下。我们需要引入一些免费的推理变量（就像在中做的那样）[`replace_regions_in_mir`]）。这个例子没有使用产生的确切区域，但它（希望）足以让这个想法得以实现。

```rust,ignore
fn foo<'a, 'b>(x: &'a /* '#1 */ usize) -> &'b /* '#3 */ usize {
    x // '#2, location L1
}
```

一些符号：`'#1`，`'#3`，和`'#2`表示参数，返回值和表达式的通用区域`x`， 分别。另外，我将调用表达式的位置`x` `L1`。

所以现在我们可以使用活动约束来获得以下起点：

| 区域 | 内容 |
| --- | --- |
| “＃1 |  |
| “＃2 | `L1` |
| “＃3 | `L1` |

现在我们使用outlives约束来扩展每个区域。具体来说，我们知道`'#2: '#3`...

| 区域 | 内容 |
| --- | --- |
| “＃1 | `L1` |
| “＃2 | `L1, end('#3) // add contents of '#3 and end('#3)` |
| “＃3 | `L1` |

......和`'#1: '#2`所以......

| 区域 | 内容 |
| --- | --- |
| “＃1 | `L1, end('#2), end('#3) // add contents of '#2 and end('#2)` |
| “＃2 | `L1, end('#3)` |
| “＃3 | `L1` |

现在，我们需要检查没有区域太大（在这种情况下我们没有任何类型测试来检查）。请注意`'#1`现在包含`end('#3)`，但我们没有`where`条款或暗示必然会这样说`'a: 'b`......那是一个错误！

### 一些细节

该[`RegionInferenceContext`]type包含进行推理所需的所有信息，包括来自的通用区域[`replace_regions_in_mir`]以及为每个区域计算的约束。它是在我们计算活度约束之后构建的。

以下是结构的一些字段：

-   [`constraints`]：包含所有outlives约束。
-   [`liveness_constraints`]：包含所有活动限制。
-   [`universal_regions`]：包含`UniversalRegions`归来的[`replace_regions_in_mir`]。
-   [`universal_region_relations`]：包含已知的关于普遍区域的关系。例如，如果我们有一个where子句`'a: 'b`，当借用检查实现（在调用者处检查）时，假定该关系为真，所以`universal_region_relations`会包含`'a:
    'b`。
-   [`type_tests`]：包含对推理后必须检查的类型的一些约束（例如：`T: 'a`）。
-   [`closure_bounds_mapping`]：用于将区域约束从闭包传播回到闭包的创建者。

[`constraints`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.constraints

[`liveness_constraints`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.liveness_constraints

[`universal_regions`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.universal_regions

[`universal_region_relations`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.universal_region_relations

[`type_tests`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.type_tests

[`closure_bounds_mapping`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#structfield.closure_bounds_mapping

TODO：我们应该讨论其他任何领域吗？SCC怎么样？

好了，现在我们已经构建了一个`RegionInferenceContext`，我们可以推断。这是通过调用[`solve`]关于上下文的方法。这就是我们打电话的地方[`propagate_constraints`]然后检查生成的类型测试和通用区域，如上所述。

[`propagate_constraints`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#method.propagate_constraints

[`check_type_tests`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#method.check_type_tests

[`check_universal_regions`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/region_infer/struct.RegionInferenceContext.html#method.check_universal_regions

## 关闭

当我们检查类型测试和通用区域时，如果我们处于封闭体中，我们可能会遇到一个我们无法证明的约束！但是，必要的约束实际上可能成立（我们还不知道）。因此，如果我们在闭包内，我们只收集我们尚未证明的所有约束并返回它们。稍后，当我们借用检查创建闭包的MIR节点时，我们也可以检查这些约束是否成立。那时，如果我们无法证明他们持有，我们会报告错误。

## 占位符和宇宙

（本节描述尚未登陆的正在进行的工作。）

我们不时要推断我们无法具体了解的地区。例如，考虑这个程序：

```rust,ignore
// A function that needs a static reference
fn foo(x: &'static u32) { }

fn bar(f: for<'a> fn(&'a u32)) {
       // ^^^^^^^^^^^^^^^^^^^ a function that can accept **any** reference
    let x = 22;
    f(&x);
}

fn main() {
    bar(foo);
}
```

这个程序不应该打字检查：`foo`需要一个静态引用的参数，和`bar`希望得到一个接受的功能**任何**引用（例如，它可以在其堆栈上调用它）。但*怎么样*我们拒绝它吗？*为什么*？

### 子类型和占位符

当我们打字检查`main`，特别是电话`bar(foo)`，我们将结束像这样的子类型关系：

```text
fn(&'static u32) <: for<'a> fn(&'a u32)
----------------    -------------------
the type of `foo`   the type `bar` expects
```

我们通过获取超类型中绑定的变量并替换它们来处理这种子类型[普遍量化](../appendix/background.html#quantified)代表，写得像`!1`。我们将这些地区称为“占位区域” - 它们基本上代表“某些未知区域”。

一旦我们完成了替换，我们就有以下关系：

```text
fn(&'static u32) <: fn(&'!1 u32)
```

这里的关键思想是这个未知区域`'!1`与任何其他地区无关。因此，如果我们能够证明子类型关系是真的`'!1`那么它应该适用于任何地区，这就是我们想要的。

那么让我们来看看下一步会发生什么。要检查两个函数是否是子类型，我们检查它们的参数是否具有所需的关系（fn参数是[逆变](../appendix/background.html#variance)，所以我们在这里左右交换）：

```text
&'!1 u32 <: &'static u32
```

根据参考的基本子类型规则，如果是这样的话`'!1: 'static`。那就是 - 如果“某个未知的地区`!1`“生活更久`'static`。现在，这个*威力*是真的 - 毕竟，`'!1`可能`'static`- 但我们没有*知道*这是真的。所以这应该会产生错误（最终）。

### 什么是宇宙

在上一节中，我们介绍了占位符区域的概念，并将其表示出来`!1`。我们称这个号码`1`该**宇宙指数**。“宇宙”的概念是它是一组在某种类型或某些点范围内的名称。宇宙形成一棵树，每个孩子都用一些新的名字扩展其父母。所以**根宇宙**概念上包含全局名称，例如生命周期`'static`或类型`i32`。在编译器中，我们还将泛型类型参数放入此根Universe（在这个意义上，不仅有一个根Universe，而是每个项目一个）。所以考虑这个功能`bar`：

```rust,ignore
struct Foo { }

fn bar<'a, T>(t: &'a T) {
    ...
}
```

在这里，根宇宙将由生命周期组成`'static`和`'a`。事实上，虽然我们专注于这里的生命周期，但我们可以将相同的概念应用于类型，在这种情况下类型`Foo`和`T`将在根宇宙中（以及其他全局类型，如`i32`）。基本上，根Universe包含所有名称[显得自由](../appendix/background.html#free-vs-bound)在体内`bar`。

现在让我们来看看吧`bar`稍微加一个变量`x`：

```rust,ignore
fn bar<'a, T>(t: &'a T) {
    let x: for<'b> fn(&'b u32) = ...;
}
```

在这里，名字`'b`不是根宇宙的一部分。相反，当我们“进入”这个`for<'b>`（例如，通过用占位符替换它），我们将创建根的子Universe，我们称之为U1：

```text
U0 (root universe)
│
└─ U1 (child universe)
```

这个想法是这个子宇宙U1用一个新名称扩展了根宇宙U0，我们用它的宇宙号来识别它：`!1`。

现在让我们来看看吧`bar`再加一个变量，`y`：

```rust,ignore
fn bar<'a, T>(t: &'a T) {
    let x: for<'b> fn(&'b u32) = ...;
    let y: for<'c> fn(&'b u32) = ...;
}
```

当我们进入*这*类型，我们将再次创建一个新的宇宙，我们称之为`U2`. 它的父代将是根宇宙，而U1将是它的兄弟：

```text
U0 (root universe)
│
├─ U1 (child universe)
│
└─ U2 (child universe)
```

这意味着，在u2中，我们可以从u0或u2来命名，但不能从u1来命名。

**给存在变量一个宇宙。**既然我们有了universe的概念，我们就可以使用它来扩展类型检查器和防止非法名称泄漏的功能。我们的想法是给每个推论（存在的）变量——无论它是一个类型还是一个生命——一个宇宙。然后，该变量的值只能引用该宇宙中可见的名称。例如，在U0中创建了一个生存期变量，那么它就不能被赋值为`!1`或`!2`因为这些名字在宇宙U0中是不可见的。

**用计数器表示宇宙。**您可能会惊讶地发现编译器没有跟踪完整的宇宙树。相反，它只保留一个计数器——为了确定一个宇宙是否可以看到另一个宇宙，它只检查索引是否更大。例如，U2可以看到U0，因为2大于等于0。但U0看不到U2，因为0>=2为假。

我们怎么能摆脱这个？这不意味着我们将允许U2也看到U1吗？答案是，是的，我们会，**如果这个问题出现了**.  但是由于我们的类型检查器等的结构，这是不可能发生的。为了让宇宙U1中发生的事情与U2中发生的事情“交流”，它们必须有一个共同的共享推理变量X。因为u1中的所有内容都只限于u1及其子元素，所以推断变量x必须在u0中。因为x在u0中，它不能命名u1（或u2）中的任何东西。通过使用一种通用的“逻辑”示例，这可能是最容易看到的：

```text
exists<X> {
   forall<Y> { ... /* Y is in U1 ... */ }
   forall<Z> { ... /* Z is in U2 ... */ }
}
```

这里，两个孔交互的唯一方法是通过x，但是当声明x时，y和z都不在范围内，因此它的值不能引用它们中的任何一个。

### Universe和占位符区域元素

但是这个错误是从哪里来的呢？事情就是这样发生的。在构造区域推理上下文时，我们可以从类型推理上下文中分辨出存在多少占位符变量（即`InferCtxt`有一个内部计数器）。对于每一个变量，我们创建一个对应的通用区域变量`!n`以及“区域元素”`placeholder(n)`. 这对应于“一些未知的其他元素集”。价值`!n`是`{placeholder(n)}`.

同时，我们也给每个存在变量一个**宇宙**（也取自`InferCtxt`）此Universe确定哪些占位符元素可能出现在其值中：例如，Universe U3中的变量可以命名`placeholder(1)`，`placeholder(2)`，和`placeholder(3)`，但不是`placeholder(4)`.请注意，推理变量的宇宙控制哪些区域元素**可以**出现在其值中；不表示区域元素**将**出现。

### 占位符和异常值约束

在区域推理引擎中，异常值约束的形式如下：

```text
V1: V2 @ P
```

在哪里？`V1`和`V2`是区域指数，因此映射到某个区域变量（可能是普遍量化的或存在量化的）。这个`P`控制流程图中有一个“点”；对于本节来说并不重要。这个变量将有一个宇宙，所以我们称之为那些宇宙`U(V1)`和`U(V2)`分别。（实际上，我们唯一关心的是`U(V1)`。）

当我们遇到这个约束时，通常的过程是从`P`.只要我们走的节点存在，我们就一直走下去。`value(V2)`我们将这些节点添加到`value(V1)`.如果我们到达一个返回点，我们会添加`end(X)`元素。那部分保持不变。

但是那时*之后*我们要迭代占位符`placeholder(x)`V2中的元素（每个元素必须对`U(V2)`但是我们可以假设这是真的，我们不需要检查它）。我们必须确保`value(V1)`突出显示这些占位符元素中的每一个。

现在有两种可能发生的方法。首先，如果`U(V1)`能看到宇宙`x`（即，`x <= U(V1)`，然后我们可以添加`placeholder(x)`到`value(V1)`然后完成。但是如果没有，那么我们就必须估计：我们可能不知道元素的集合是什么`placeholder(x)`表示，但我们应该能够计算**上界**B代表IT–一些地区B比其他地区更长寿`placeholder(x)`. 现在，我们就用`'static`为此（因为它比任何事情都要长寿）–在未来，我们有时可以在这里变得更聪明（事实上，在其他环境中我们已经有了这样做的代码）。此外，由于`'static`在根宇宙U0中，我们知道所有变量都能看到它，所以基本上如果我们发现`value(V2)`包含`placeholder(x)`对于某些宇宙`x`那个`V1`看不见，我们就强行`V1`到`'static`.

### 扩展“通用区域”检查

在传播了所有约束之后，NLL区域推断有一个最终检查，它将检查每个通用区域的最终计算值，并检查它们是否“太大”。在我们的例子中，我们将遍历每个占位符区域并检查它是否包含*只有*这个`placeholder(u)`元素的寿命是已知的。（稍后，我们可能会知道两个占位符区域之间存在关系，并将这些关系考虑在内，正如我们从fn签名中对通用区域所做的那样。）

换句话说，“通用区域”检查可以被认为是检查约束，例如：

```text
{placeholder(1)}: V1
```

在哪里？`{placeholder(1)}`就像一个常量集，v1是我们用来表示`!1`区域。

## 回到我们的例子

好的，到目前为止还不错。现在，让我们来看看第一个例子会发生什么：

```text
fn(&'static u32) <: fn(&'!1 u32) @ P  // this point P is not imp't here
```

区域推理引擎将创建这样的区域元素域：

```text
{ CFG; end('static); placeholder(1) }
    ---  ------------  ------- from the universe `!1`
    |    'static is always in scope
    all points in the CFG; not especially relevant here
```

它将始终创建两个通用变量，一个表示`'static`一个代表`'!1`. 让我们称它们为vs和v1。它们的初始值如下：

```text
Vs = { CFG; end('static) } // it is in U0, so can't name anything else
V1 = { placeholder(1) }
```

从上面的子类型约束来看，我们有一个异常值约束，比如

```text
'!1: 'static @ P
```

为了处理这一点，我们将把v1的值增加到包括所有vs：

```text
Vs = { CFG; end('static) }
V1 = { CFG; end('static), placeholder(1) }
```

在这一点上，约束传播是完整的，因为所有的异常值关系都得到了满足。然后，我们将转到代码的“检查通用区域”部分，该部分将测试没有任何通用区域增长过大。

在这种情况下，`V1` *做*变得太大了——我们不知道它会活得太久。`end('static)`也不是cfg中的任何一个——所以我们会报告一个错误。

## 另一个例子

这种子类型关系怎么样？

```text
for<'a> fn(&'a u32, &'a u32)
    <:
for<'b, 'c> fn(&'b u32, &'c u32)
```

这里，我们将用占位符替换父类型中的绑定区域，如前所述，从而生成：

```text
for<'a> fn(&'a u32, &'a u32)
    <:
fn(&'!1 u32, &'!2 u32)
```

然后我们用一个存在于宇宙U2中的存在论来实例化左侧的变量，得出如下结论：（`?n`是存在变量的符号）：

```text
fn(&'?3 u32, &'?3 u32)
    <:
fn(&'!1 u32, &'!2 u32)
```

然后我们进一步细分：

```text
&'!1 u32 <: &'?3 u32
&'!2 u32 <: &'?3 u32
```

更进一步，放弃我们的地区限制：

```text
'!1: '?3
'!2: '?3
```

注意，在这种情况下，两者都是`'!1`和`'!2`必须超越变量`'?3`但是变量`'?3`不会被迫比其他任何东西都长寿。因此，它只是作为空元素集开始和结束，因此类型检查在这里成功。

（这会让你有点惊讶。当我第一次意识到这一点时，我很惊讶。我们是说如果我们是一个**需要它的两个参数具有相同的区域**，我们可以接受**具有两个不同区域的参数**. 这似乎直觉上是不健全的。但事实上，这很好，正如我在[这个问题][ohdeargoditsallbroken]关于铁锈问题的追踪器。原因是，即使我们使用两个不同生命周期的参数来调用，这两个生命周期也有一些交集（调用本身），交集可以是`'a`作为我们争论的共同生命。- NMASSAKIS

[ohdeargoditsallbroken]: https://github.com/rust-lang/rust/issues/32330#issuecomment-202536977

## 最后实例

让我们来看最后一个例子。我们将扩展前一个类型以具有返回类型：

```text
for<'a> fn(&'a u32, &'a u32) -> &'a u32
    <:
for<'b, 'c> fn(&'b u32, &'c u32) -> &'b u32
```

尽管看起来与前一个例子非常相似，但本例将得到一个错误。这很好：问题是，我们已经从一个承诺返回其两个论点之一的fn转变为一个承诺返回第一个论点的fn。那是不健全的。让我们看看结果如何。

首先，我们用占位符替换父类型中的绑定区域：

```text
for<'a> fn(&'a u32, &'a u32) -> &'a u32
    <:
fn(&'!1 u32, &'!2 u32) -> &'!1 u32
```

然后我们用存在主义（在U2中）实例化子类型：

```text
fn(&'?3 u32, &'?3 u32) -> &'?3 u32
    <:
fn(&'!1 u32, &'!2 u32) -> &'!1 u32
```

现在我们创建子类型关系：

```text
&'!1 u32 <: &'?3 u32 // arg 1
&'!2 u32 <: &'?3 u32 // arg 2
&'?3 u32 <: &'!1 u32 // return type
```

最后是离群关系。这里，让v1、v2和v3作为我们分配给的变量`!1`，`!2`和`?3`分别：

```text
V1: V3
V2: V3
V3: V1
```

这些变量将具有以下初始值：

```text
V1 in U1 = {placeholder(1)}
V2 in U2 = {placeholder(2)}
V3 in U2 = {}
```

现在因为`V3: V1`约束，我们必须添加`placeholder(1)`进入之内`V3`（事实上，从`V3`我们得到：

```text
V3 in U2 = {placeholder(1)}
```

那么我们有这个约束`V2: V3`所以我们不得不放大`V2`包括`placeholder(1)`（它也可以看到）：

```text
V2 in U2 = {placeholder(1), placeholder(2)}
```

现在约束传播完成了，但是当我们检查异常值关系时，我们发现`V2`包括这个新元素`placeholder(1)`，所以我们报告了一个错误。

## 借阅检查器错误

托多：我们应该讨论如何从这些分析的结果中产生错误。
