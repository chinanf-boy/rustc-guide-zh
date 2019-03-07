# 移动路径

实际上，在局部变量的粒度上跟踪初始化是不够的。Rust还允许我们在字段粒度上执行移动和初始化：

```rust,ignore
fn foo() {
    let a: (Vec<u32>, Vec<u32>) = (vec![22], vec![44]);
    
    // a.0 and a.1 are both initialized
    
    let b = a.0; // moves a.0
    
    // a.0 is not initializd, but a.1 still is

    let c = a.0; // ERROR
    let d = a.1; // OK
}
```

为了处理这个问题，我们在**移动路径**. 一[`MovePath`]表示用户可以初始化、移动等的位置，例如，有一个移动路径表示局部变量`a`，并且有一个移动路径表示`a.0`.  移动路径大致符合[`Place`]但它们的索引方式使我们能够更有效地进行移动分析。

[`movepath`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MovePath.html

[`place`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/mir/enum.Place.html

## 移动路径索引

虽然有一个[`MovePath`]数据结构，它们从不直接引用。相反，所有的代码都会传递*指数*类型的[`MovePathIndex`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/indexes/struct.MovePathIndex.html). 如果需要获取有关移动路径的信息，可以将此索引与[`move_paths`领域`MoveData`][move_paths]. 例如，要转换[`MovePathIndex`] `mpi`进入密尔[`Place`]，您可以访问[`MovePath::place`]像这样的字段：

```rust,ignore
move_data.move_paths[mpi].place
```

[move_paths]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MoveData.html#structfield.move_paths

[`movepath::place`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MovePath.html#structfield.place

[`movepathindex`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/indexes/struct.MovePathIndex.html

## 建立移动路径

我们在mir borrow check中首先要做的事情之一是构建一组移动路径。这是作为[`MoveData::gather_moves`]功能。此函数使用名为[`Gatherer`]走在和平号上看看[`Place`]在中访问。对于每一个这样的[`Place`]，它构造一个对应的[`MovePathIndex`]. 它还记录移动/初始化特定移动路径的时间/位置，但我们将在后面的部分中讨论到这一点。

[`gatherer`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/builder/struct.Gatherer.html

[`movedata::gather_moves`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MoveData.html#method.gather_moves

### 非法移动路径

我们并没有为**每一个** [`Place`]它被利用了。尤其是，如果从[`Place`]那么就不需要[`MovePathIndex`].一些例子：

-   不能从静态变量移动，因此我们不创建[`MovePathIndex`]对于静态变量。
-   您不能移动数组的单个元素，因此如果我们`foo: [String; 3]`，将没有移动路径`foo[1]`.
-   您不能从借用的引用内部移动，因此如果我们`foo: &String`，将没有移动路径`*foo`.

这些规则是由[`move_path_for`]函数，它转换[`Place`]变成[`MovePathIndex`]--在刚才讨论的错误情况下，函数返回`Err`.这反过来意味着我们不必费心跟踪这些地方是否被初始化（这会降低开销）。

[`move_path_for`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/builder/struct.Gatherer.html#method.move_path_for

## 查找移动路径

如果你有[`Place`]你想把它转换成[`MovePathIndex`]，您可以使用[`MovePathLookup`]在中找到的结构[`rev_lookup`]领域[`MoveData`].有两种不同的方法：

[`movepathlookup`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MovePathLookup.html

[`rev_lookup`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MoveData.html#structfield.rev_lookup

-   [`find_local`]，这需要[`mir::Local`]表示局部变量。这是更简单的方法，因为我们**总是**创建一个[`MovePathIndex`]对于每个局部变量。
-   [`find`]，这需要一个任意的[`Place`]. 这种方法使用起来有点烦人，因为我们没有[`MovePathIndex`]对于**每一个** [`Place`]（正如我们刚才在“非法移动路径”部分讨论的那样）。因此，[`find`]返回A[`LookupResult`]指示它能够找到的最接近的路径（例如`foo[1]`，它可能只返回`foo`）

[`find`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MovePathLookup.html#method.find

[`find_local`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/struct.MovePathLookup.html#method.find_local

[`mir::local`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/mir/struct.Local.html

[`lookupresult`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/move_paths/enum.LookupResult.html

## 交叉引用

如上所述，移动路径存储在一个大向量中，并通过它们引用[`MovePathIndex`]. 然而，在这个向量中，它们也被构造成一棵树。例如，如果你有[`MovePathIndex`]对于`a.b.c`，可以转到其父移动路径`a.b`. 您还可以迭代所有子路径：因此，从`a.b`，您可以迭代以查找路径`a.b.c`（这里，您将迭代**实际引用**在源头，不是全部**可能的**可能被引用的路径）。例如，在[`has_any_child_of`]函数，它检查数据流结果是否包含给定移动路径的值（例如，`a.b`）或该移动路径的任何子级（例如，`a.b.c`）

[`place`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/mir/enum.Place.html

[`has_any_child_of`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/dataflow/at_location/struct.FlowAtLocation.html#method.has_any_child_of
