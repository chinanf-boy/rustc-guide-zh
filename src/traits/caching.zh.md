# 缓存和相关的细微考虑

一般来说，我们尝试缓存特性选择的结果。这是一个有点复杂的过程。部分原因是我们希望能够缓存结果，即使特性引用中的所有类型都不完全已知。在这种情况下，特性选择过程也可能影响类型变量，因此我们不仅要能够缓存*结果*选择过程，但是*重播*它对类型变量的影响。

## 一个例子

缓存如何工作的高级概念是，我们首先用占位符版本替换所有未绑定的推理变量。因此，如果我们有一个特性参考`usize : Foo<$t>`在哪里`$t`是一个未绑定的推理变量，我们可以将其替换为`usize : Foo<$0>`在哪里`$0`是占位符类型。然后我们会在缓存中查找这个。

如果我们找到一个命中，命中将告诉我们选择过程中要采取的下一步（例如，apply impl 22或apply where子句`X : Foo<Y>`）

另一方面，如果没有击中，我们需要通过[选择过程]白手起家。假设，我们得出的结论是，唯一可能的IMPL是这个IMPL，带有def id 22：

[selection process]: ./resolution.html#selection

```rust,ignore
impl Foo<isize> for usize { ... } // Impl #22
```

然后我们将在缓存中记录`usize : Foo<$0> => ImplCandidate(22)`. 下一步我们会[确认] `ImplCandidate(22)`这将（作为副作用）统一`$t`具有`isize`.

[confirm]: ./resolution.html#confirmation

现在，在以后的某个时候，我们可能会来看看`usize :
Foo<$u>`. 当替换为占位符时，这将产生`usize : Foo<$0>`和以前一样，因此缓存查找将成功，从而`ImplCandidate(22)`. 我们会确认`ImplCandidate(22)`这将（作为副作用）统一`$u`具有`isize`.

## WHERE子句和本地与全局缓存

一个微妙的交互作用是特征查找的结果将根据作用域中子句的位置而变化。因此，我们实际上*二*缓存、本地缓存和全局缓存。本地缓存附加到[`ParamEnv`]以及附加到[`tcx`]. 只要结果可能取决于作用域中的WHERE子句，我们就使用本地缓存。通过该方法确定要使用的缓存`pick_candidate_cache`在里面`select.rs`. 目前，我们使用一个非常简单、保守的规则：如果作用域中有任何where子句，那么我们使用本地缓存。我们曾经试图画出更细微的区别，但这导致了一系列令人讨厌和奇怪的错误，比如[α22019#]和[α18290#]. 这个简单的规则看起来非常安全，并且仍然保持很高的命中率（编译rustc时大约95%）。

**托多**看起来像`pick_candidate_cache`不再存在。一般来说，这一部分仍然准确吗？

[`paramenv`]: ../param_env.html

[`tcx`]: ../ty.html

[#18290]: https://github.com/rust-lang/rust/issues/18290

[#22019]: https://github.com/rust-lang/rust/issues/22019
