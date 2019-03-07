# MIR类型检查

借入支票的一个关键组成部分是[MIR型检查](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/nll/type_check/index.html)。该检查遍历MIR并进行完整的“类型检查” - 与您在任何其他语言中找到的类型相同。在执行此类型检查的过程中，我们还会发现适用于该程序的区域约束。

TODO  - 进一步详细说明？也许？:)
