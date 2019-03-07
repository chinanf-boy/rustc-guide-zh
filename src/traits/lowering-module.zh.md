# Rustc中的下降模块

中描述的计划条款[下沉规则](./lowering-rules.html)节实际创建于[`rustc_traits::lowering`][lowering]模块。

[lowering]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_traits/lowering/

## 这个`program_clauses_for`查询

主要入口点是`program_clauses_for` [查询]它——给定一个def id——生成一组粉笔程序子句。这些查询使用[专用单元测试机制，如下所述](#unit-tests).  查询是在`DefId`它标识诸如特征、IMPL或关联项定义之类的内容。然后生成并返回程序子句的向量。

[query]: ../query.html

<a name="unit-tests"></a>

## 单元测试

单元测试位于[`src/test/ui/chalkify`][chalkify]. 一个很好的例子是[这个`lower_impl`测试][lower_impl]. 在写这篇文章的时候，它看起来是这样的：

```rust,ignore
#![feature(rustc_attrs)]

trait Foo { }

#[rustc_dump_program_clauses] //~ ERROR Implemented(T: Foo) :-
impl<T: 'static> Foo for T where T: Iterator<Item = i32> { }

fn main() {
    println!("hello");
}
```

这个`#[rustc_dump_program_clauses]`注释可以附加到任何具有def-id的对象上。（它需要`rustc_attrs`然后编译器将调用`program_clauses_for`查询该项，并发出编译器错误，以转储生成的子句。这些错误只存在于单元测试中，因为我们可以利用标准[用户界面测试]检查它们的机制。在这种情况下，有一个`//~ ERROR Implemented`注释是有意最小化的（它只需要是错误的前缀），但是[STDRR文件]包含完整的详细信息：

```text
error: Implemented(T: Foo) :- ProjectionEq(<T as std::iter::Iterator>::Item == i32), TypeOutlives(T \
: 'static), Implemented(T: std::iter::Iterator), Implemented(T: std::marker::Sized).
  --> $DIR/lower_impl.rs:15:1
   |
LL | #[rustc_dump_program_clauses] //~ ERROR Implemented(T: Foo) :-
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error
```

[chalkify]: https://github.com/rust-lang/rust/tree/master/src/test/ui/chalkify

[lower_impl]: https://github.com/rust-lang/rust/tree/master/src/test/ui/chalkify/lower_impl.rs

[the stderr file]: https://github.com/rust-lang/rust/tree/master/src/test/ui/chalkify/lower_impl.stderr

[ui test]: ../tests/adding.html#guide-to-the-ui-tests
