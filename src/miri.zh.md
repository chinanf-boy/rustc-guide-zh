# 米里

米里**密尔** **我**interpreter）是一个虚拟机，用于执行mir而不编译为机器代码。通常通过以下方式调用`tcx.const_eval`.

如果你从一个常量开始

```rust
const FOO: usize = 1 << 12;
```

在使用常量或将常量放入元数据之前，rustc实际上不会调用任何东西。

一旦你有了一个像

```rust,ignore
type Foo = [u8; FOO - 42];
```

编译器需要计算出数组的长度，然后才能创建使用该类型的项（局部变量、常量、函数参数等）。

要获取（在本例中为空）参数环境，可以调用`let param_env = tcx.param_env(length_def_id);`. 这个`GlobalId`需要的是

```rust,ignore
let gid = GlobalId {
    promoted: None,
    instance: Instance::mono(length_def_id),
};
```

调用`tcx.const_eval(param_env.and(gid))`现在将触发创建数组长度表达式的mir。和平号看起来像这样：

```mir
const Foo::{{initializer}}: usize = {
    let mut _0: usize;                   // return pointer
    let mut _1: (usize, bool);

    bb0: {
        _1 = CheckedSub(const Unevaluated(FOO, Slice([])), const 42usize);
        assert(!(_1.1: bool), "attempt to subtract with overflow") -> bb1;
    }

    bb1: {
        _0 = (_1.0: usize);
        return;
    }
}
```

在评估之前，虚拟内存位置（在本例中，基本上是`vec![u8; 4]`或`vec![u8; 8]`）用于存储评估结果。

在评估开始时，`_0`和`_1`是`ConstValue::Scalar(Scalar::Undef)`. 当初始化`_1`是调用的，其值`FOO`常量是必需的，并触发对`tcx.const_eval`，此处不显示。如果FOO的评估成功，42将减去其值。`4096`结果存储在`_1`作为`ConstValue::ScalarPair(Scalar::Bytes(4054), Scalar::Bytes(0))`. 对的第一部分是计算值，第二部分是bool，如果发生溢出，则为true。

下一个语句断言所说的布尔值是`0`. 如果断言失败，它的错误消息将用于报告编译时错误。

因为它没有失败，`ConstValue::Scalar(Scalar::Bytes(4054))`是存储在虚拟内存中之前分配的。`_0`总是直接指那个位置。

在评估完成后，虚拟内存分配被放入`TyCtxt`.将来对相同常量的评估实际上不会调用miri，而是从内部分配中提取值。

这个`tcx.const_eval`函数还有一个附加功能：它不会返回`ByRef(interned_allocation_id)`，但A`Scalar(computed_value)`如果可能的话。这使得使用结果更加方便，因为不需要执行进一步的查询来获得像`usize`.

## 数据结构

Miri的核心数据结构可以在[librustc/mir/解释](https://github.com/rust-lang/rust/blob/master/src/librustc/mir/interpret).这主要是错误枚举和`ConstValue`和`Scalar`类型。A`ConstValue`可以是`Scalar`（单人间）`Scalar`），请`ScalarPair`（二`Scalar`s，通常是胖指针或两个元素元组）或`ByRef`，用于其他任何内容并引用虚拟分配。可以通过上的方法访问这些分配`tcx.interpret_interner`.

如果需要数值结果，可以使用`unwrap_usize`（对任何不能代表为`u64`）或`assert_usize`结果是`Option<u128>`屈服于`Scalar`如果可能的话。

## 分配

MIRI分配是内存的字节序列或`Instance`对于函数指针。字节序列还可以包含将一组字节标记为指向另一个分配的指针的重定位。重定位时的实际字节指的是另一个分配中的偏移量。

这些分配的存在使引用和原始指针具有指向的内容。没有全局线性堆在其中进行分配，但是每个分配（无论是用于局部变量、静态还是（将来的）堆分配）都会获得自己的小内存，其大小正好是所需的大小。因此，如果您有一个指向局部变量分配的指针`a`，您所能做的任何操作（无论多么不安全）都不可能将所说的指针更改为指向`b`.

## 解释

尽管持续评估的主要切入点是`tcx.const_eval`查询，中有其他函数[librustc-mir/const-eval.rs.公司](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/const_eval/index.html)允许访问`ConstValue`（`ByRef`否则）。您不应该访问`Allocation`直接将其转换为编译目标（目前只有llvm）。

MIRI首先为正在评估的当前常量创建一个虚拟堆栈帧。常量和没有参数的函数本质上没有区别，除了常量在编写本指南时不允许使用局部（命名）变量。

堆栈帧由`Frame`键入[librustc_mir/explote/eval_context.rs](https://github.com/rust-lang/rust/blob/master/src/librustc_mir/interpret/eval_context.rs)包含所有局部变量的内存（`None`在评估开始时）。每个帧都引用根常量或后续对`const fn`. 对另一个常量的计算只需调用`tcx.const_eval`从而产生一个全新的独立堆栈帧。

框架只是一个`Vec<Frame>`，无法实际引用`Frame`即使可怕的恶作剧是通过不安全的代码完成的。唯一能被引用的内存是`Allocation`s.

米莉现在打电话给`step`方法（in）[librustc_mir/解释/步骤rs](https://github.com/rust-lang/rust/blob/master/src/librustc_mir/interpret/step.rs)）直到它返回一个错误或者没有更多的语句可以执行。每个语句现在都将初始化或修改本地变量或本地变量引用的虚拟内存。这可能需要计算其他常量或静态，这些常量或静态只是递归调用`tcx.const_eval`.
