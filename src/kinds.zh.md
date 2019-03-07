# 种类

一`ty::subst::Kind<'tcx>`表示类型系统中的某个实体：类型（`Ty<'tcx>`）寿命`ty::Region<'tcx>`或常数`ty::Const<'tcx>`）`Kind`用于为具体参数执行泛型参数替换，例如使用类型参数显式调用带有泛型参数的函数时。替换用[`Subst`类型](#subst)如下所述。

## `Subst`

`ty::subst::Subst<'tcx>`直观上只是`Kind<'tcx>`s，作为从泛型参数到具体参数（如类型、生存期和常量）的替换的有序列表。

例如，给定`HashMap<K, V>`有两个类型参数，`K`和`V`例如，参数的实例化`HashMap<i32, u32>`，将由替换表示`&'tcx [tcx.types.i32, tcx.types.u32]`.

`Subst`为实例化给定项定义的替换提供各种方便方法，通常应使用这些方法，而不是显式构造此类替换切片。

## `Kind`

实际`Kind`结构针对空间进行了优化，将类型、生存期或常量存储为包含标识其类型的掩码的内部指针（最低2位）。除非你和`Subst`具体来说，您通常不需要处理`Kind`而是利用保险箱[`UnpackedKind`](#unpackedkind)抽象化。

## `UnpackedKind`

As `Kind`它本身不是类型安全的，`UnpackedKind`枚举为处理类型提供了更方便和安全的接口。安`UnpackedKind`可以转换为原始`Kind`使用`Kind::from()`（或者简单地`.into()`当上下文清晰时）。如前所述，子状态列表存储原始`Kind`因此，在处理它们之前，最好将它们转换为`UnpackedKind`首先。这是通过调用`.unpack()`方法。

```rust,ignore
// An example of unpacking and packing a kind.
fn deal_with_kind<'tcx>(kind: Kind<'tcx>) -> Kind<'tcx> {
    // Unpack a raw `Kind` to deal with it safely.
    let new_kind: UnpackedKind<'tcx> = match kind.unpack() {
        UnpackedKind::Type(ty) => { /* ... */ }
        UnpackedKind::Lifetime(lt) => { /* ... */ }
        UnpackedKind::Const(ct) => { /* ... */ }
    };
    // Pack the `UnpackedKind` to store it in a substitution list.
    new_kind.into()
}
```
