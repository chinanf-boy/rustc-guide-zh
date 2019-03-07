# 持续评估

持续评估是在编译时计算值的过程。对于特定项目（常量/静态/数组长度），这在项目的MIR被借用检查和优化之后发生。在许多情况下，尝试const评估项目将首次触发其MIR的计算。

突出的例子是

-   初始化者`static`
-   数组长度
    -   需要知道保留堆栈或堆空间
-   Enum变体判别式
    -   需要知道是为了防止两个变体具有相同的判别式
-   模式
    -   需要知道检查重叠模式

此外，通过在编译时预先计算复杂操作并仅存储结果，可以使用常量评估来减少运行时的工作负载或二进制大小。

可以通过调用来进行持续评估`const_eval`查询`TyCtxt`。

该`const_eval`查询需要一个[`ParamEnv`](./param_env.html)评估常数的环境（例如，使用常数的函数）和a`GlobalId`。该`GlobalId`是由一个`Instance`指一个常数或静态或一个`Instance`函数的函数和函数的索引`Promoted`表。

持续评估返回a`Result`无论是错误，还是常量的最简单表示。“最简单”意味着如果它可以表示为整数或胖指针，它将直接产生值（通过`ConstValue::Scalar`要么`ConstValue::ScalarPair`），而不是指[`miri`](./miri.html)虚拟内存分配（通过`ConstValue::ByRef`）。这意味着`const_eval`函数不能用于创建计算常量或静态的miri指针。如果需要，您需要直接使用中的函数[SRC / librustc_mir / const_eval.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/const_eval/index.html)。
