# 形态检查

WF检查的任务是检查防锈程序中的各种声明的格式是否正确。这是隐含界限的基础，部分出于这个原因，这种检查可能非常微妙！例如，我们必须确保每个IMPL都证明了特性上声明的WF条件。

对于Rust程序中的每个声明，我们将生成一个逻辑目标，并尝试使用我们在[下沉规则](./lowering-rules.md)章。如果我们能证明这一点，我们就说这个结构是成形的。如果没有，我们会向用户报告一个错误。

良好的形式检查发生在[`src/rules/wf.rs`][wf]粉笔模组。阅读本章之后，您可能会发现在[`src/rules/wf/test.rs`][wf_test]子模块。

新型的WF检查还没有在RustC中实现。

[wf]: https://github.com/rust-lang-nursery/chalk/blob/master/src/rules/wf.rs

[wf_test]: https://github.com/rust-lang-nursery/chalk/blob/master/src/rules/wf/test.rs

我们在这里提供了每个生锈声明生成目标的完整参考。

除了在降低规则一章中介绍的符号之外，我们还将介绍另一个符号：在检查声明的WF时，我们通常必须证明所有出现的类型都是格式良好的，除了我们一直假定为WF的类型参数。因此，我们将使用以下符号：对于类型`SomeType<...>`我们定义`InputTypes(SomeType<...>)`是出现在`SomeType<...>`包括`SomeType<...>`本身。

实例：

-   `InputTypes((u32, f32)) = [u32, f32, (u32, f32)]`
-   `InputTypes(Box<T>) = [Box<T>]`（假设`T`是类型参数）
-   `InputTypes(Box<Box<T>>) = [Box<T>, Box<Box<T>>]`

我们还扩展了`InputTypes`以自然的方式标注到WHERE子句。因此，例如`InputTypes(A0: Trait<A1,...,An>)`是联盟`InputTypes(A0)`，`InputTypes(A1)`…`InputTypes(An)`.

# 类型定义

给定常规类型定义：

```rust,ignore
struct Type<P...> where WC_type {
    field1: A1,
    ...
    fieldn: An,
}
```

我们生成以下目标，表示其良好的形式条件：

```text
forall<P...> {
    if (FromEnv(WC_type)) {
        WellFormed(InputTypes(WC_type)) &&
            WellFormed(InputTypes(A1)) &&
            ...
            WellFormed(InputTypes(An))
    }
}
```

在英语中：假设类型上定义的where子句保持不变，则证明类型定义中出现的每个类型都是格式良好的。

一些例子：

```rust,ignore
struct OnlyClone<T> where T: Clone {
    clonable: T,
}
// The only types appearing are type parameters: we have nothing to check,
// the type definition is well-formed.
```

````rust,ignore
struct Foo<T> where T: Clone {
    foo: OnlyClone<T>,
}
// The only non-parameter type which appears in this definition is
// `OnlyClone<T>`. The generated goal is the following:
// ```
// forall<T> {
//     if (FromEnv(T: Clone)) {
//          WellFormed(OnlyClone<T>)
//     }
// }
// ```
// which is provable.
````

````rust,ignore
struct Bar<T> where <T as Iterator>::Item: Debug {
    bar: i32,
}
// The only non-parameter types which appear in this definition are
// `<T as Iterator>::Item` and `i32`. The generated goal is the following:
// ```
// forall<T> {
//     if (FromEnv(<T as Iterator>::Item: Debug)) {
//          WellFormed(<T as Iterator>::Item) &&
//               WellFormed(i32)
//     }
// }
// ```
// which is not provable since `WellFormed(<T as Iterator>::Item)` requires
// proving `Implemented(T: Iterator)`, and we are unable to prove that for an
// unknown `T`.
//
// Hence, this type definition is considered illegal. An additional
// `where T: Iterator` would make it legal.
````

# 特征定义

给出一般特征定义：

```rust,ignore
trait Trait<P1...> where WC_trait {
    type Assoc<P2...>: Bounds_assoc where WC_assoc;
}
```

我们生成以下目标：

```text
forall<P1...> {
    if (FromEnv(WC_trait)) {
        WellFormed(InputTypes(WC_trait)) &&

            forall<P2...> {
                if (FromEnv(WC_assoc)) {
                    WellFormed(InputTypes(Bounds_assoc)) &&
                        WellFormed(InputTypes(WC_assoc))
                }
            }
    }
}
```

在特性定义中没有太多要验证的内容。我们只想证明在特征定义中出现的类型是形成良好的，前提是不同的where子句都有。

一些例子：

````rust,ignore
trait Foo<T> where T: Iterator, <T as Iterator>::Item: Debug {
    ...
}
// The only non-parameter type which appears in this definition is
// `<T as Iterator>::Item`. The generated goal is the following:
// ```
// forall<T> {
//     if (FromEnv(T: Iterator), FromEnv(<T as Iterator>::Item: Debug)) {
//         WellFormed(<T as Iterator>::Item)
//     }
// }
// ```
// which is provable thanks to the `FromEnv(T: Iterator)` assumption.
````

````rust,ignore
trait Bar {
    type Assoc<T>: From<<T as Iterator>::Item>;
}
// The only non-parameter type which appears in this definition is
// `<T as Iterator>::Item`. The generated goal is the following:
// ```
// forall<T> {
//     WellFormed(<T as Iterator>::Item)
// }
// ```
// which is not provable, hence the trait definition is considered illegal.
````

````rust,ignore
trait Baz {
    type Assoc<T>: From<<T as Iterator>::Item> where T: Iterator;
}
// The generated goal is now:
// ```
// forall<T> {
//     if (FromEnv(T: Iterator)) {
//         WellFormed(<T as Iterator>::Item)
//     }
// }
// ```
// which is now provable.
````

# 暗示

现在，我们给自己一个对上述特征的一般性的恳求：

```rust,ignore
impl<P1...> Trait<A1...> for SomeType<A2...> where WC_impl {
    type Assoc<P2...> = SomeValue<A3...> where WC_assoc;
}
```

注意，在这里，`WC_assoc`与特征声明中关联类型定义上定义的WHERE子句相同，*除了*特性中的类型参数由IMPL提供的值替换（见下面的示例）。不能添加新的WHERE子句。如果你想强调你实际上并不依赖于WHERE子句的事实，你可以省略它。

举例说明：

```rust,ignore
trait Foo<T> {
    type Assoc where T: Clone;
}

struct OnlyClone<T: Clone> { ... }

impl<U> Foo<Option<U>> for () {
    // We substitute type parameters from the trait by the ones provided
    // by the impl, that is instead of having a `T: Clone` where clause,
    // we have an `Option<U>: Clone` one.
    type Assoc = OnlyClone<Option<U>> where Option<U>: Clone;
}

impl<T> Foo<T> for i32 {
    // I'm not using the `T: Clone` where clause from the trait, so I can
    // omit it.
    type Assoc = u32;
}

impl<T> Foo<T> for f32 {
    type Assoc = OnlyClone<Option<T>> where Option<T>: Clone;
    //                                ^^^^^^^^^^^^^^^^^^^^^^
    //                                this where clause does not exist
    //                                on the original trait decl: illegal
}
```

> 所以在Rust中，相关类型的条款起作用*确切地*就像特性方法上的WHERE子句一样：在IMPL中，我们必须用IMPL提供的值替换特性中的参数，如果不需要它们，我们可以省略它们，但是我们不能添加新的WHERE子句。

现在，让我们看看为这个通用IMPL生成的目标：

```text
forall<P1...> {
    // Well-formedness of types appearing in the impl
    if (FromEnv(WC_impl), FromEnv(InputTypes(SomeType<A2...>: Trait<A1...>))) {
        WellFormed(InputTypes(WC_impl)) &&

            forall<P2...> {
                if (FromEnv(WC_assoc)) {
                        WellFormed(InputTypes(SomeValue<A3...>))
                }
            }
    }

    // Implied bounds checking
    if (FromEnv(WC_impl), FromEnv(InputTypes(SomeType<A2...>: Trait<A1...>))) {
        WellFormed(SomeType<A2...>: Trait<A1...>) &&

            forall<P2...> {
                if (FromEnv(WC_assoc)) {
                    WellFormed(SomeValue<A3...>: Bounds_assoc)
                }
            }
    }
}
```

这是最复杂的目标。和往常一样，首先，假设存在各种各样的where子句，我们证明IMPL中出现的每种类型都是格式良好的，***除了***出现在IMPL头中的类型`SomeType<A2...>: Trait<A1...>`. 相反，我们*假定*这些类型形成良好（因此`if (FromEnv(InputTypes(SomeType<A2...>: Trait<A1...>)))`条件）。这是隐含边界建议的一部分，因此我们可以依赖于根据定义编写的边界，例如`SomeType<A2...>`类型（我们不需要重复这些界限）。

> 注意，我们不需要检查出现在`WC_assoc`因为我们已经在特征decl中这样做了（它们只是重复了一些值的替换，我们已经假定这些值是形成良好的）

接下来，仍然假设IMPL上的WHERE子句`WC_impl`保持和输入类型`SomeType<A2...>`我们证明`WellFormed(SomeType<A2...>: Trait<A1...>)`保持。也就是说，我们要证明`SomeType<A2...>`验证`Trait`定义（见）[本分部](./implied-bounds.md#co-inductiveness-of-wellformed)）

最后，假设关联类型上的WHERE子句`WC_assoc`等等，我们证明`WellFormed(SomeValue<A3...>: Bounds_assoc)`保持。再说一遍，我们不仅要证明`Implemented(SomeValue<A3...>: Bounds_assoc)`以及所有可能传递的事实`Bounds_assoc`. 我们必须这样做，因为我们允许在关联类型上使用隐含的边界：如果`FromEnv(SomeType: Trait)`在我们的环境中，降低规则一章表明我们能够`FromEnv(<SomeType as Trait>::Assoc: Bounds_assoc)`不知道它的确切价值`<SomeType as Trait>::Assoc`是。

生成目标的一些示例：

````rust,ignore
// Trait Program Clauses

// These are program clauses that come from the trait definitions below
// and that the trait solver can use for its reasonings. I'm just restating
// them here so that we have them in mind.

trait Copy { }
// This is a program clause that comes from the trait definition above
// and that the trait solver can use for its reasonings. I'm just restating
// it here (and also the few other ones coming just after) so that we have
// them in mind.
// `WellFormed(Self: Copy) :- Implemented(Self: Copy).`

trait Partial where Self: Copy { }
// ```
// WellFormed(Self: Partial) :-
//     Implemented(Self: Partial) &&
//     WellFormed(Self: Copy).
// ```

trait Complete where Self: Partial { }
// ```
// WellFormed(Self: Complete) :-
//     Implemented(Self: Complete) &&
//     WellFormed(Self: Partial).
// ```

// Impl WF Goals

impl<T> Partial for T where T: Complete { }
// The generated goal is:
// ```
// forall<T> {
//     if (FromEnv(T: Complete)) {
//         WellFormed(T: Partial)
//     }
// }
// ```
// Then proving `WellFormed(T: Partial)` amounts to proving
// `Implemented(T: Partial)` and `Implemented(T: Copy)`.
// Both those facts can be deduced from the `FromEnv(T: Complete)` in our
// environment: this impl is legal.

impl<T> Complete for T { }
// The generated goal is:
// ```
// forall<T> {
//     WellFormed(T: Complete)
// }
// ```
// Then proving `WellFormed(T: Complete)` amounts to proving
// `Implemented(T: Complete)`, `Implemented(T: Partial)` and
// `Implemented(T: Copy)`.
//
// `Implemented(T: Complete)` can be proved thanks to the
// `impl<T> Complete for T` blanket impl.
//
// `Implemented(T: Partial)` can be proved thanks to the
// `impl<T> Partial for T where T: Complete` impl and because we know
// `T: Complete` holds.

// However, `Implemented(T: Copy)` cannot be proved: the impl is illegal.
// An additional `where T: Copy` bound would be sufficient to make that impl
// legal.
````

````rust,ignore
trait Bar { }

impl<T> Bar for T where <T as Iterator>::Item: Bar { }
// We have a non-parameter type appearing in the where clauses:
// `<T as Iterator>::Item`. The generated goal is:
// ```
// forall<T> {
//     if (FromEnv(<T as Iterator>::Item: Bar)) {
//         WellFormed(T: Bar) &&
//             WellFormed(<T as Iterator>::Item: Bar)
//     }
// }
// ```
// And `WellFormed(<T as Iterator>::Item: Bar)` is not provable: we'd need
// an additional `where T: Iterator` for example.
````

````rust,ignore
trait Foo { }

trait Bar {
    type Item: Foo;
}

struct Stuff<T> { }

impl<T> Bar for Stuff<T> where T: Foo {
    type Item = T;
}
// The generated goal is:
// ```
// forall<T> {
//     if (FromEnv(T: Foo)) {
//         WellFormed(T: Foo).
//     }
// }
// ```
// which is provable.
````

````rust,ignore
trait Debug { ... }
// `WellFormed(Self: Debug) :- Implemented(Self: Debug).`

struct Box<T> { ... }
impl<T> Debug for Box<T> where T: Debug { ... }

trait PointerFamily {
    type Pointer<T>: Debug where T: Debug;
}
// `WellFormed(Self: PointerFamily) :- Implemented(Self: PointerFamily).`

struct BoxFamily;

impl PointerFamily for BoxFamily {
    type Pointer<T> = Box<T> where T: Debug;
}
// The generated goal is:
// ```
// forall<T> {
//     WellFormed(BoxFamily: PointerFamily) &&
//
//     if (FromEnv(T: Debug)) {
//         WellFormed(Box<T>: Debug) &&
//             WellFormed(Box<T>)
//     }
// }
// ```
// `WellFormed(BoxFamily: PointerFamily)` amounts to proving
// `Implemented(BoxFamily: PointerFamily)`, which is ok thanks to our impl.
//
// `WellFormed(Box<T>)` is always true (there are no where clauses on the
// `Box` type definition).
//
// Moreover, we have an `impl<T: Debug> Debug for Box<T>`, hence
// we can prove `WellFormed(Box<T>: Debug)` and the impl is indeed legal.
````

````rust,ignore
trait Foo {
    type Assoc<T>;
}

struct OnlyClone<T: Clone> { ... }

impl Foo for i32 {
    type Assoc<T> = OnlyClone<T>;
}
// The generated goal is:
// ```
// forall<T> {
//     WellFormed(i32: Foo) &&
//        WellFormed(OnlyClone<T>)
// }
// ```
// however `WellFormed(OnlyClone<T>)` is not provable because it requires
// `Implemented(T: Clone)`. It would be tempting to just add a `where T: Clone`
// bound inside the `impl Foo for i32` block, however we saw that it was
// illegal to add where clauses that didn't come from the trait definition.
````
