# 降低

降低步骤将AST转换为[希尔](hir.html). 这意味着，如果许多结构与类型分析或类似的语法不可知分析无关，那么它们将被删除。此类结构的示例包括但不限于

-   括号
    -   删除而不替换，树结构使顺序显式
-   `for`循环和`while (let)`循环
    -   转换为`loop` + `match`还有一些`let`绑定
-   `if let`
    -   转换为`match`
-   通用的`impl Trait`
    -   转换为通用参数（但带有一些标志，以知道用户没有写入这些参数）
-   存在的`impl Trait`
    -   转换为虚拟`existential type`宣言

降低需要维护几个不变量，以便不触发正常签入`src/librustc/hir/map/hir_id_validator.rs`：

1.  一`HirId`必须在创建时使用。所以如果你使用`lower_node_id`你*必须*使用结果`NodeId`或`HirId`（两者都很好，因为`NodeId`S在`HIR`检查是否存在`HirId`s）
2.  降A`HirId`必须在*拥有*项目。这意味着你需要使用`with_hir_id_owner`如果您正在创建一个项目的部分，而不是当前正在降低的部分。例如，在降低存在主义`impl Trait`
3.  一`NodeId`这将被放置到一个HIR结构必须降低，即使它`HirId`未被使用。打电话`let _ = self.lower_node_id(node_id);`完全合法。
4.  如果创建的新节点不存在于`AST`你*必须*为他们创建新的ID。这是通过调用`next_id`方法，它同时生成`NodeId`同时自动降低它，这样您也可以`HirId`.

如果您正在创建新的`DefId`S，因为每个`DefId`需要有相应的`NodeId`，建议添加这些`NodeId`S到`AST`所以你不必在下降过程中生成新的。这有一个优势，就是创造一种方法来找到`DefId`通过它`NodeId`. 如果下降需要这个`DefId`在多个地方，无法生成新的`NodeId`在所有这些地方，因为你也会得到一个新的`DefId`然后。用一个`NodeId`从`AST`这不是问题。

拥有`NodeId`也允许`DefCollector`生成`DefId`而不是降低必须做的飞行。集中`DefId`在一个地方生成可以更容易地重构和推理。
