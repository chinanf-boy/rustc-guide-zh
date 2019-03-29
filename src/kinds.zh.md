# Kinds

一个`ty::subst::Kind<'tcx>`表示类型系统中的某个实体：一个类型（`Ty<'tcx>`），一个生命周期(`ty::Region<'tcx>`或常数`ty::Const<'tcx>`）。`Kind`用于为具体参数，执行泛型参数替换，例如，使用类型参数显式调用，带有泛型参数的函数时。用[`Subst`类型](#subst)代表替换，如下所述。

## `Subst`

`ty::subst::Subst<'tcx>`直观上，只是一个`Kind<'tcx>`切片，作为从泛型参数到具体参数（如类型、生命周期和常量）替换的有序列表。

例如，给出一个`HashMap<K, V>`，它有两个类型参数，`K`和`V`。一个参数实例，例如：`HashMap<i32, u32>`，将由`&'tcx [tcx.types.i32, tcx.types.u32]`替换表示。

`Subst`为所给予的定义项（实例）替换，提供各种方便方法，通常应使用这些方法，而不是显式构造替换切片。

## `Kind`

真正的`Kind`结构，针对空间进行了优化，将类型、生命周期或常量存储为内部指针，其包含标识其类型的掩码（最低 2 位）。除非你与`Subst`的具体实现一起工作，不然的话，通常你不需要处理`Kind`，不过，要用安全[`UnpackedKind`](#unpackedkind)抽象代替。

## `UnpackedKind`

正如，`Kind`本身不是类型安全的，`UnpackedKind`枚举为处理类型，提供了更方便和安全的接口。一个`UnpackedKind`可以用`Kind::from()`，转换为一个原始`Kind`（或者简单用`.into()`，当上下文清晰时）。如前所述，替换列表存储原始`Kind`，因此，在处理它们之前，最好先将它们转换为`UnpackedKind`。这是通过调用`.unpack()`方法。

```rust,ignore
// 一个示例，关于 unpacking 和 packing 一个 kind.
fn deal_with_kind<'tcx>(kind: Kind<'tcx>) -> Kind<'tcx> {
    // Unpack(解包) 一个原始 `Kind` ，处理它的安全性.
    let new_kind: UnpackedKind<'tcx> = match kind.unpack() {
        UnpackedKind::Type(ty) => { /* ... */ }
        UnpackedKind::Lifetime(lt) => { /* ... */ }
        UnpackedKind::Const(ct) => { /* ... */ }
    };
    // Pack(打包) `UnpackedKind` ，把它存储在替换列表
    new_kind.into()
}
```
