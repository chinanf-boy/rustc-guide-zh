### 这个`#[test]`属性

如今，Rust程序员依靠一个内置的属性`#[test]`. 您所要做的就是将一个函数标记为一个测试，并包含一些这样的断言：

```rust,ignore
#[test]
fn my_test() {
  assert!(2+2 == 4);
}
```

当使用`rustc --test`或`cargo test`，它将生成一个可执行文件，可以运行此文件和任何其他测试函数。这种测试方法允许测试以一种有机的方式与代码共存。您甚至可以将测试放在私有模块中：

```rust,ignore
mod my_priv_mod {
  fn my_priv_func() -> bool {}

  #[test]
  fn test_priv_func() {
    assert!(my_priv_func());
  }
}
```

因此，私人物品可以很容易地进行测试，而不必担心如何将它们暴露在任何类型的外部测试设备中。这是铁锈试验的人机工程学的关键。然而，从语义上讲，这相当奇怪。什么样的`main`如果这些测试不可见，函数是否调用它们？究竟是什么`rustc --test`干什么？

`#[test]`在编译器的[`libsyntax`机箱][libsyntax]. 本质上，它是一个奇特的宏，可以用3个步骤重写箱子：

#### 步骤1:重新导出

如前所述，测试可以存在于私有模块中，因此我们需要一种将它们公开给主函数的方法，而不破坏任何现有的代码。为此，`libsyntax`将创建名为`__test_reexports`递归地重新导出测试。此扩展将上述示例转换为：

```rust,ignore
mod my_priv_mod {
  fn my_priv_func() -> bool {}

  pub fn test_priv_func() {
    assert!(my_priv_func());
  }

  pub mod __test_reexports {
    pub use super::test_priv_func;
  }
}
```

现在，我们的测试可以访问为`my_priv_mod::__test_reexports::test_priv_func`. 对于更深的模块结构，`__test_reexports`将重新导出包含测试的模块，因此`a::b::my_test`变成`a::__test_reexports::b::__test_reexports::my_test`. 虽然这个过程看起来相当安全，但是如果存在`__test_reexports`模块？答案是：没什么。

为了解释，我们需要理解[AST如何表示标识符][ident]. 每个函数、变量、模块等的名称不是作为字符串存储的，而是作为不透明的[符号][symbol]它本质上是每个标识符的ID号。编译器保留一个单独的哈希表，允许我们在必要时（如打印语法错误时）恢复符号的可读名称。当编译器生成`__test_reexports`模块，它为标识符生成一个新的符号，因此当编译器生成`__test_reexports`可以和你亲手写一个名字，它不会和你共享一个符号。该技术防止代码生成过程中的名称冲突，是RIST宏卫生的基础。

#### 步骤2：线束生成

既然我们的测试可以从箱子的底部进行，我们需要对它们做些什么。`libsyntax`生成这样的模块：

```rust,ignore
pub mod __test {
  extern crate test;
  const TESTS: &'static [self::test::TestDescAndFn] = &[/*...*/];

  #[main]
  pub fn main() {
    self::test::test_static_main(TESTS);
  }
}
```

虽然这种转换很简单，但它让我们对测试实际上是如何运行有了很多了解。这些测试被聚合到一个数组中并传递给一个名为`test_static_main`. 我们会回到正题`TestDescAndFn`但现在，关键的是有一个箱子叫[`test`][test]这是Rust核心的一部分，它实现了所有用于测试的运行时。`test`的接口不稳定，因此与之交互的唯一稳定方式是通过`#[test]`宏。

#### 步骤3：测试对象生成

如果您以前在Rust中编写过测试，那么您可能熟悉测试函数中可用的一些可选属性。例如，测试可以用注释`#[should_panic]`如果我们期望测试会引起恐慌。看起来像这样：

```rust,ignore
#[test]
#[should_panic]
fn foo() {
  panic!("intentional");
}
```

这意味着我们的测试不仅仅是简单的函数，它们还有配置信息。`test`将此配置数据编码到名为[`TestDesc`][testdesc]. 对于箱子中的每个测试功能，`libsyntax`将分析其属性并生成`TestDesc`实例。然后它结合了`TestDesc`和测试函数`TestDescAndFn`结构，`test_static_main`继续运转。对于给定的测试，生成的`TestDescAndFn`实例看起来是这样的：

```rust,ignore
self::test::TestDescAndFn{
  desc: self::test::TestDesc{
    name: self::test::StaticTestName("foo"),
    ignore: false,
    should_panic: self::test::ShouldPanic::Yes,
    allow_fail: false,
  },
  testfn: self::test::StaticTestFn(||
    self::test::assert_test_result(::crate::__test_reexports::foo())),
}
```

一旦我们构建了这些测试对象的数组，它们将通过步骤2中生成的线束传递给测试运行程序。

### 检查生成的代码

在夜间生锈的地方，有一面不稳定的旗子叫`unpretty`可用于在宏展开后打印出模块源：

```bash
$ rustc my_mod.rs -Z unpretty=hir
```

[test]: https://doc.rust-lang.org/test/index.html

[testdesc]: https://doc.rust-lang.org/test/struct.TestDesc.html

[symbol]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.Ident.html

[ident]: https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.Ident.html

[erfc]: https://github.com/rust-lang/rfcs/blob/master/text/2318-custom-test-frameworks.md

[libsyntax]: https://github.com/rust-lang/rust/tree/master/src/libsyntax
