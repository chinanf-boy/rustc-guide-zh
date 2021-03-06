# 附录C：术语表

编译器使用了许多...特殊的缩写和东西。本词汇表试图列出它们，并为您提供一些指导，以便更好地理解它们。

| 术语 | 含义 |
| --- | --- |
| AST | 语法包生成的抽象语法树;非常密切地反映用户语法。 |
| 粘合剂 | “binder”是声明变量或类型的地方;例如，`<T>`是泛型类型参数的绑定器`T`在`fn foo<T>(..)`，和\|`a`\|`...`是参数的活页夹`a`。看到[背景章节了解更多](./background.html#free-vs-bound) |
| 绑定变量 | “绑定变量”是在表达式/术语中声明的变量。例如，变量`a`在封闭期间受到约束\|`a`\|`a * 2`。看到[背景章节了解更多](./background.html#free-vs-bound) |
| 代码生成 | 将MIR转换为LLVM IR的代码。 |
| 代码单元 | 当我们生成LLVM IR时，我们将Rust代码分组为多个codegen单元。这些单元中的每一个都由LLVM彼此独立地处理，从而实现并行性。它们也是增量再利用的单位。 |
| 完整性 | 完整性是类型理论中的技术术语。完整性意味着每个类型安全的程序也会进行类型检查。兼顾完整性和完整性非常困难，通常健全性更重要。（见“健全”）。 |
| 控制流图 | 表示程序的控制流程;看到[背景章节了解更多](./background.html#cfg) |
| CTFE | 编译时功能评估。这是编译器评估的能力`const fn`在编译时。这是编译器常量评估系统的一部分。（[看更多](../const-eval.html)） |
| cx | 我们倾向于使用“cx”作为上下文的缩写。也可以看看`tcx`，`infcx`等 |
| DAG | 在编译期间使用有向非循环图来跟踪查询之间的依赖关系。（[看更多](../queries/incremental-compilation.html)） |
| 数据流分析 | 静态分析，确定程序控制流中每个点的属性是正确的;看到[背景章节了解更多](./background.html#dataflow) |
| DefId | 标识定义的索引（参见`librustc/hir/def_id.rs`）。唯一标识一个`DefPath`。 |
| 双指针 | 带有其他元数据的指针。有关更多信息，请参阅“胖指针”。 |
| 滴胶 | （内部）编译器生成的指令，用于处理调用析构函数（`Drop`）用于数据类型。 |
| DST | 动态大小的类型。编译器无法静态知道内存大小的类型（例如`str`要么`[u8]`）。这种类型没有实现`Sized`并且不能在堆栈上分配。它们只能作为结构中的最后一个字段出现。它们只能在指针后面使用（例如`&str`要么`&[u8]`）。 |
| 早期生命 | 在其定义站点替换的生命周期区域。绑定在一个项目中`Generics`并用a代替`Substs`。对比**晚生活**。（[看更多](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/enum.RegionKind.html#bound-regions)） |
| 空类型 | 看“无人居住的类型”。 |
| 胖指针 | 带有某个值的地址的双字值，以及使用该值所需的一些其他信息。Rust包含两种“胖指针”：对切片的引用和特征对象。对切​​片的引用携带切片的起始地址及其长度。特征对象携带一个值的地址和一个指向适合该值的特征实现的指针。“胖指针”也被称为“宽指针”和“双指针”。 |
| 自由变量 | “自由变量”是指未在表达式或术语中约束的变量;看到[背景章节了解更多](./background.html#free-vs-bound) |
| “GCX | 全球舞台的生命周期（[看更多](../ty.html)） |
| 仿制药 | 在类型或项上定义的泛型类型参数集 |
| HIR | 高级IR，通过降低和消除AST（[看更多](../hir.html)） |
| HirId | 通过将def-id与“定义内偏移”组合来识别HIR中的特定节点。 |
| HIR地图 | 可通过tcx.hir访问的HIR地图允许您快速导航HIR并在各种形式的标识符之间进行转换。 |
| 冰 | 内部编译错误。当编译器崩溃时。 |
| ICH | 增量编译哈希。ICH用作HIR和crate元数据等指纹，用于检查是否已进行更改。这在增量编译中非常有用，可以查看包的某些部分是否已更改并应重新编译。 |
| 推理变量 | 在进行类型或区域推理时，“推理变量”是一种表示您要推断的特殊类型/区域。想想代数中的X.例如，如果我们试图推断程序中变量的类型，我们创建一个推理变量来表示该未知类型。 |
| infcx | 推理上下文（见`librustc/infer`） |
| 实习生 | 实习是指存储某些常用的常数数据，例如字符串，然后通过标识符（例如，a。）来引用数据`Symbol`）而不是数据本身，以减少内存使用。 |
| IR | 中级代表。编译器中的通用术语。在编译期间，代码从原始源（ASCII文本）转换为各种IR。在Rust中，这些主要是HIR，MIR和LLVM IR。每个IR都非常适合某些计算。例如，MIR非常适合借用检查程序，LLVM IR非常适合codegen，因为LLVM接受它。 |
| IRLO | `IRLO`要么`irlo`有时用作缩写[internals.rust-lang.org](https://internals.rust-lang.org)。 |
| 项目 | 语言中的一种“定义”，例如static，const，use语句，模块，struct等。具体来说，这对应于`Item`类型。 |
| 郎项目 | 表示语言本身固有概念的项目，例如特殊的内置特征`Sync`和`Send`;或表示诸如的操作的特征`Add`;或编译器调用的函数。（[看更多](https://doc.rust-lang.org/1.9.0/book/lang-items.html)） |
| 晚生活 | 在其呼叫站点替换的生命周期区域。绑定在HRTB中并由编译器中的特定函数替换，例如`liberate_late_bound_regions`。对比**早期生命**。（[看更多](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/enum.RegionKind.html#bound-regions)） |
| 当地的箱子 | 目前正在编制的箱子。 |
| LTO | 链接时优化。LLVM提供的一组优化，在最终二进制文件链接之前发生。这些包括优化，例如删除最终程序中从未使用过的函数。*ThinLTO*是LTO的一种变体，旨在提高可扩展性和效率，但可能会牺牲一些优化。您还可以阅读关于“FatLTO”的Rust回购中的问题，这是给予非瘦LTO的爱好昵称。LLVM文档：[这里][lto]和[这里][thinlto] |
| [LLVM] | （实际上不是首字母缩略词：P）一个开源编译器后端。它接受LLVM IR并输出本机二进制文件。然后，各种语言（例如Rust）可以实现输出LLVM IR的编译器前端，并使用LLVM编译到LLVM支持的所有平台。 |
| memoize的 | memoization是存储（纯）计算结果（如纯函数调用）的过程，以避免将来重复它们。这通常是执行速度和内存使用之间的权衡。 |
| MIR | 在类型检查之后创建的中级IR，由borrowck和codegen使用（[看更多](../mir/index.html)） |
| 美里 | 用于持续评估的MIR翻译（[看更多](../miri.html)） |
| 正常化 | 转换为更规范形式的一般术语，但在rustc的情况下通常是指[关联类型规范化](../traits/associated-types.html#normalize) |
| newtype | “newtype”是其他类型的包装器（例如，`struct Foo(T)`是一个“新类型”`T`）。这通常用在Rust中，为索引提供更强的类型。 |
| NLL | [非词汇生命](../borrow_check/region_inference.html)，Rust的借用系统的扩展，使其基于控制流图。 |
| node-id或NodeId | 标识AST或HIR中特定节点的索引;逐渐被淘汰并取而代之`HirId`。 |
| 义务 | 必须通过特质系统证明的东西（[看更多](../traits/resolution.html)） |
| 投影 | “相对路径”的一般术语，例如`x.f`是“场投影”，并且`T::Item`是一个[“关联类型投影”](../traits/goals-and-clauses.html#trait-ref) |
| 提升常量 | 从函数中提取并提升到静态范围的常量；请参见[本节](../mir/index.html#promoted)了解更多详细信息。 |
| 供应商 | 执行查询的函数（[查看更多](../query.html)） |
| 量化的 | 在数学或逻辑中，存在主义和普遍的量化被用来问“是否有任何类型的t是真的？”或者“所有T型都是这样吗？”；见[更多背景章节](./background.html#quantified) |
| 查询 | 也许在编译过程中有一些子计算（[查看更多](../query.html)） |
| 区域 | “一生”的另一个术语，常用于文献和借阅检查中。 |
| 肋骨 | 名称解析程序中的数据结构，用于跟踪名称的单个作用域。（[查看更多](../name-resolution.html)） |
| 自评税 | 编译器会话，它存储整个编译过程中使用的全局数据。 |
| 边桌 | 因为AST和HIR一旦创建就不可变，所以我们通常以哈希表的形式携带关于它们的额外信息，并通过特定节点的ID进行索引。 |
| 印记 | 类似于关键字，但完全由非字母数字标记组成。例如，`&`是一个参考信号。 |
| 占位符 | **注意：skolemization已被占位符弃用**处理“for all”类型的子类型的方法（例如，`for<'a> fn(&'a u32)`）以及解决更高等级的特征界限（例如，`for<'a> T: Trait<'a>`）见[关于占位符和Universes的章节](../borrow_check/region_inference.html#placeholders-and-universes)了解更多详细信息。 |
| 安定性 | 稳健性是类型论中的一个技术术语。大致上，如果一个类型系统是健全的，那么如果一个程序类型检查，它是类型安全的；也就是说，我永远不能（在安全锈中）强制一个值进入错误类型的变量。（见“完整性”）。 |
| 跨度 | 用户源代码中的一个位置，主要用于错误报告。它们就像类固醇上的文件名/行号/列元组：它们携带一个起点/终点，还跟踪宏扩展和编译器删除。同时打包成几个字节（实际上，它是表中的索引）。有关更多信息，请参见SPAN数据类型。 |
| 潜艇 | 给定泛型类型或项的替换（例如`i32`，`u32`在里面`HashMap<i32, u32>`） |
| tcx | “键入上下文”，编译器的主要数据结构（[查看更多](../ty.html)） |
| TCX | 当前活动推理上下文的生存期（[查看更多](../ty.html)） |
| 性状参考 | 特性的名称以及一组合适的输入类型/生存期。（[查看更多](../traits/goals-and-clauses.html#trait-ref)） |
| 令牌 | 解析的最小单位。词法分析后生成令牌（[查看更多](../the-parser.html)） |
| [热释光] | 线程本地存储。变量可以定义为每个线程都有自己的副本（而不是共享变量的所有线程）。这与LLVM有一些交互。并非所有平台都支持TLS。 |
| 反式 | 将mir转换为llvm ir的代码。重命名为codegen。 |
| 性状参考 | 其类型参数的特性和值（[查看更多](../ty.html)） |
| ty | 类型的内部表示法（[查看更多](../ty.html)） |
| 超滤膜 | 通用函数调用语法。用于调用方法的明确语法（[查看更多](../type-checking.html)） |
| 无人居住型 | 一种具有*不*价值观。这与zst不同，zst只有1个值。无人居住类型的一个例子是`enum Foo {}`，它没有变量，因此永远无法创建。编译器可以将处理无人居住类型的代码视为死代码，因为没有这样的值需要操作。`!`（never类型）是无人居住的类型。无人居住的类型也被称为“空类型”。 |
| UVAR | 从闭包外部由闭包捕获的变量。 |
| 方差 | 方差决定对泛型类型/生存期参数的更改如何影响子类型；例如，如果`T`是一个亚型`U`然后`Vec<T>`是一个亚型`Vec<U>`因为`Vec`是*协变的*在其泛型参数中。见[背景章节](./background.html#variance)更一般的解释。见[差异章节](../variance.html)有关类型检查如何处理差异的说明。 |
| 宽指针 | 带有附加元数据的指针。更多信息请参见“胖指针”。 |
| ZST | 零大小类型。值大小为0字节的类型。自从`2^0 = 1`，此类类型只能有一个值。例如，`()`（单位）是zst。`struct Foo;`也是一个ZST。编译器可以围绕zst进行一些很好的优化。 |

[llvm]: https://llvm.org/

[lto]: https://llvm.org/docs/LinkTimeOptimization.html

[thinlto]: https://clang.llvm.org/docs/ThinLTO.html

[tls]: https://llvm.org/docs/LangRef.html#thread-local-storage-models
