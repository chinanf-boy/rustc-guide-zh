# 降层

降层步骤将 AST 转换为[HIR](hir.zh.html)。 这意味着，许多结构将被删除，若是它们与类型分析，或不可知的类似语法分析无关。此类结构的示例包括，但不限于：

- 括号
  - 删除，不替换，因树结构让顺序更显式
- `for`循环和`while (let)`循环
  - 转换为`loop` + `match`，还有一些`let`绑定
- `if let`
  - 转换为`match`
- 一般的`impl Trait`
  - 转换为泛型参数（但带有一些标志，以知道用户没有写入这些参数）
- 存在的(Existential) `impl Trait`
  - 转换为一个虚拟`existential type`声明

降层需要维护几个不变量，以便不触发在`src/librustc/hir/map/hir_id_validator.rs`的正常检查：

1. 一个`HirId`必须在创建时使用。所以，如果你使用`lower_node_id`，那你*必须*使用`NodeId`或`HirId`结果（两者都形，因为在`HIR`中`NodeId`们，会检查`HirId`们是否存在）
2. 降层一个`HirId`，必须在*所有权*项的作用域中完成。这意味着，如果您正在创建一个项的部分，而不是当前正在降层的部分，那你需要使用`with_hir_id_owner`。这种情况，会发生在降层 existential `impl Trait`期间。
3. 一个`NodeId`，将被放置到一个 HIR 结构，因此必须降层，即使它的`HirId`未被使用。调用`let _ = self.lower_node_id(node_id);`完全合法。
4. 如果创建的新节点不存在于`AST`，你*必须*为他们创建新的 ID。这是通过调用`next_id`方法，完成的，它即生成一个`NodeId`，同时自动帮你降层它，这样您也可以得到这个`HirId`.

如果您正在创建新的`DefId`，又因为每个`DefId`需要有相应的`NodeId`，建议添加这些`NodeId`到`AST`，所以你不必在降层过程中生成新的。创造一种方法，通过某某的`NodeId`，找到它的`DefId`，会有好处。 如果降层过程中，多个地方需要这个`DefId`，你不会也不能在所有这些地方，生成新的`NodeId`，因为你之后也会得到一个新的`DefId`。使用来自`AST`的一个`NodeId`，从不是问题。

拥有的`NodeId`，也允许`DefCollector`生成`DefId`，而不是降层必须做的。把`DefId`生成集中在一个地方，可以更容易地重构，也更合理。
