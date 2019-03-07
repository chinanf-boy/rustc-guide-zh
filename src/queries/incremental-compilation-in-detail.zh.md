# 详细的增量编译

增量编译方案本质上是对整个查询系统的一个非常简单的扩展。它依赖于这样一个事实：

1.  查询是纯函数——给定相同的输入，查询将始终产生相同的结果，并且
2.  查询模型在非循环图中构造编译，使各个计算之间的依赖关系显式化。

本章将解释如何使用这些属性使事情增量化，然后继续讨论版本实现问题。

# 一种增量查询评估的基本算法

如中所述[查询评估模型入门][query-model]，查询调用形成有向非循环图。下面是上一章的例子：

```ignore
  list_of_all_hir_items <----------------------------- type_check_crate()
                                                               |
                                                               |
  Hir(foo) <--- type_of(foo) <--- type_check_item(foo) <-------+
                                      |                        |
                    +-----------------+                        |
                    |                                          |
                    v                                          |
  Hir(bar) <--- type_of(bar) <--- type_check_item(bar) <-------+
```

由于从一个查询到另一个查询的每次访问都必须经过查询上下文，所以我们可以记录这些访问，从而在内存中实际构建这个依赖关系图。在启用依赖项跟踪的情况下，当编译完成时，我们知道调用了哪些查询（图的节点）和每次调用，其他哪些查询或输入已用于计算查询结果（图的边缘）。

现在假设，我们更改程序的源代码，使`bar`看起来和以前不一样。我们的目标是在重新使用所有其他查询的缓存结果的同时，只重新计算那些实际受更改影响的查询。根据依赖关系图，我们可以做到这一点。对于一个给定的查询调用，图形会准确地告诉我们计算其结果所用的数据是什么，我们只需要沿着边缘走，直到到达发生变化的地方。如果我们没有遇到任何变化，我们知道查询的结果仍然与缓存中的结果相同。

采取`type_of(foo)`从上面的调用作为示例，我们可以通过跟踪其输入的边缘来检查缓存的结果是否仍然有效。唯一的优势是`Hir(foo)`，未受更改影响的输入。所以我们知道`type_of(foo)`仍然有效。

这个故事有点不同`type_check_item(foo)`我们再次走到边缘，已经知道了`type_of(foo)`很好。然后我们到达`type_of(bar)`我们还没有检查过，所以我们走在`type_of(bar)`邂逅`Hir(bar)`哪一个*有*改变。结果是`type_of(bar)`可能会产生与缓存中的结果不同的结果，并且可以传递地是`type_check_item(foo)`可能也改变了。我们因此重新运行`type_check_item(foo)`，然后重新运行`type_of(bar)`，因为它读取的是最新版本的`Hir(bar)`.

# 基本算法的问题：误报

如果你仔细阅读上一段，你会发现上面写着`type_of(bar)` *可以*已更改，因为其中一个输入已更改。还有一种可能性，即它仍可能产生完全相同的结果。*尽管*它的输入已更改。考虑一个简单查询的示例，该查询只计算整数的符号：

```ignore
  IntValue(x) <---- sign_of(x) <--- some_other_query(x)
```

让我们说`IntValue(x)`开始作为`1000`然后设置为`2000`. 尽管`IntValue(x)`两种情况不同，`sign_of(x)`产生结果`+`在这两种情况下。

但是，如果我们遵循基本算法，`some_other_query(x)`必须（不必要地）重新计算，因为它依赖于已更改的输入。在这种情况下，变更检测会产生“假阳性”，因为它必须保守地假设`some_other_query(x)`可能会受到更改的输入的影响。

不幸的是，编译器中的实际查询中充满了这样的例子，对输入的微小更改通常可能会影响输出二进制文件的很大一部分。因此，我们必须使变更检测系统更加智能和准确。

# 提高精度：红绿算法

通过交错的变更检测和查询重新评估，可以解决“误报”问题。当试图确定某个缓存结果是否仍然有效时，我们可以检查结果是否*事实上*在我们被迫重新评估后发生了变化。

我们称此算法为红绿算法，无论好坏，因为如果我们能够证明其缓存结果仍然有效，则依赖关系图中的节点将被分配为绿色，如果重新评估结果后发现结果不同，则将被分配为红色。

红绿变化跟踪是在Try-Mark-Green算法中实现的，您已经猜到，它试图将给定节点标记为绿色：

```rust,ignore
fn try_mark_green(tcx, current_node) -> bool {

    // Fetch the inputs to `current_node`, i.e. get the nodes that the direct
    // edges from `node` lead to.
    let dependencies = tcx.dep_graph.get_dependencies_of(current_node);

    // Now check all the inputs for changes
    for dependency in dependencies {

        match tcx.dep_graph.get_node_color(dependency) {
            Green => {
                // This input has already been checked before and it has not
                // changed; so we can go on to check the next one
            }
            Red => {
                // We found an input that has changed. We cannot mark
                // `current_node` as green without re-running the
                // corresponding query.
                return false
            }
            Unknown => {
                // This is the first time we are look at this node. Let's try
                // to mark it green by calling try_mark_green() recursively.
                if try_mark_green(tcx, dependency) {
                    // We successfully marked the input as green, on to the
                    // next.
                } else {
                    // We could *not* mark the input as green. This means we
                    // don't know if its value has changed. In order to find
                    // out, we re-run the corresponding query now!
                    tcx.run_query_for(dependency);

                    // Fetch and check the node color again. Running the query
                    // has forced it to either red (if it yielded a different
                    // result than we have in the cache) or green (if it
                    // yielded the same result).
                    match tcx.dep_graph.get_node_color(dependency) {
                        Red => {
                            // The input turned out to be red, so we cannot
                            // mark `current_node` as green.
                            return false
                        }
                        Green => {
                            // Re-running the query paid off! The result is the
                            // same as before, so this particular input does
                            // not invalidate `current_node`.
                        }
                        Unknown => {
                            // There is no way a node has no color after
                            // re-running the query.
                            panic!("unreachable")
                        }
                    }
                }
            }
        }
    }

    // If we have gotten through the entire loop, it means that all inputs
    // have turned out to be green. If all inputs are unchanged, it means
    // that the query result corresponding to `current_node` cannot have
    // changed either.
    tcx.dep_graph.mark_green(current_node);

    true
}

// Note: The actual implementation can be found in
//       src/librustc/dep_graph/graph.rs
```

通过使用红绿色标记，我们可以避免在变更检测过程中出现误报的破坏性累积效应。每当以增量模式执行查询时，我们首先检查它是否已经是绿色的。如果没有，我们就跑`try_mark_green()`在上面。如果之后它仍然不是绿色的，那么我们实际上调用查询提供程序来重新计算结果。

# 现实世界：坚持是如何使一切变得复杂的

上面的部分描述了增量编译的底层算法，但是由于编译器进程在完成后退出，并将带有结果缓存的查询上下文置于遗忘状态，所以我们将数据保存到磁盘，以便下一个编译会话可以使用它。这带来了一系列全新的实施挑战：

-   查询结果缓存存储在磁盘上，因此它们不容易用于更改比较。
-   随后的编译会话将从应用了任意更改的代码的新版本开始。从全局顺序计数器生成的各种ID和索引（例如`NodeId`，`DefId`等等）可能发生了变化，使得磁盘上的持久化结果不再立即可用，因为相同的数字ID和索引可能在新的编译会话中引用了全新的内容。
-   将事物持久化到磁盘是有代价的，所以在编译会话之间不应该实际地缓存每一小块信息。固定大小的、普通的旧数据比在（反）序列化期间需要运行分支代码的复杂数据更受欢迎。

以下部分描述编译器当前如何解决这些问题。

## 稳定性问题：弥合编译会话之间的鸿沟

如前所述，各种ID（如`DefId`）由编译器以依赖于正在编译的源代码内容的方式生成。ID分配通常是确定性的，也就是说，如果两次编译完全相同的代码，相同的事情将以相同的ID结束。但是，如果有什么变化，例如在文件中间添加了一个函数，就不能保证任何东西都具有与以前相同的ID。

因此，我们不能像在内存中表示数据那样表示磁盘缓存中的数据。例如，如果我们只是存储一段类型信息，比如`TyKind::FnDef(DefId, &'tcx Substs<'tcx>)`（正如我们在记忆中所做的）然后`DefId`在新的编译会话中指向一个不同的函数，我们会遇到麻烦。

解决这个问题的方法是找到ID的“稳定”表单，这些表单在编译会话之间保持有效。对于最重要的情况，`DefId`S，这些是所谓的`DefPath`每一个`DefId`有一个对应的`DefPath`但代替数字标识`DefPath`基于已标识项的路径，例如`std::collections::HashMap`. 这样的ID的优点是它不受无关更改的影响。例如，可以将新函数添加到`std::collections`但是`std::collections::HashMap`仍然是`std::collections::HashMap`. 一`DefPath`在对源代码进行更改时是“稳定的”，而`DefId`不是。

还有`DefPathHash`它只是一个128位的哈希值`DefPath`. 两者包含相同的信息，我们主要使用`DefPathHash`因为处理起来更简单，`Copy`而且是自给自足的。

稳定标识符的这一原则用于使磁盘缓存中的数据对源代码更改具有弹性。而不是存储`DefId`我们存储`DefPathHash`当我们从缓存反序列化某些内容时，我们将`DefPathHash`到相应的`DefId`在*现在的*编译会话（这只是一个简单的哈希表查找）。

这个`HirId`，用于识别没有自己的HIR组件`DefId`，是另一个如此稳定的ID。`DefPath`和A`LocalId`那里`LocalId`识别某物（例如`hir::Expr`）在其“所有者”（例如`hir::Item`）如果所有者被移动，则`LocalId`里面的S仍然是一样的。

## 检查查询结果是否有更改：StableHash和指纹

为了进行红绿标记，我们经常需要检查查询结果是否与前一个编译会话中的结果相比发生了更改。不过，这有两个性能问题：

-   我们希望避免为了进行比较而从磁盘加载前面的结果。我们已经计算了新的结果，并将使用它。同样，从磁盘加载结果也会“污染”那些不太可能被使用的数据。
-   我们不希望将每个结果存储在磁盘缓存中。例如，将已经在上游板条箱中可用的东西保存到磁盘上会浪费精力。

编译器通过使用所谓的`Fingerprint`s.每次计算新的查询结果时，查询引擎将计算结果的128位散列值。我们称这个哈希值为`Fingerprint`查询结果的“”。散列是（并且必须）以“稳定的方式”完成的。这意味着，每当对某些内容进行哈希处理时，编译会话之间可能会发生变化（例如`DefId`，而是散列它的稳定等价物（例如`DefPath`）这就是全部`StableHash`基础设施是为。这种方式`Fingerprint`在两个不同的编译会话中计算的s仍然是可比较的。

下一步是将这些指纹与依赖关系图一起存储。这很便宜，因为指纹只是要复制的字节。将整个指纹集与依赖关系图一起加载也很便宜。

现在，当红绿色标记到达需要检查结果是否发生更改的点时，它可以将（已加载的）以前的指纹与新结果的指纹进行比较。

这种方法相当有效，但并非没有缺陷：

-   哈希冲突的可能性很小。也就是说，两个不同的结果可能具有相同的指纹，系统会错误地假设结果没有更改，从而导致错过更新。

    我们通过使用高质量的哈希函数和128位宽的哈希值来降低这种风险。由于这些措施，哈希冲突的实际风险可以忽略不计。

-   计算指纹是相当昂贵的。这是增量编译比非增量编译慢的主要原因。我们被迫使用一个好的、因而昂贵的散列函数，并且在进行散列时，我们必须将事物映射到它们的稳定等价物。

在未来，我们可能希望探索解决这个问题的不同方法。现在它是`StableHash`和`Fingerprint`.

## 新旧两部小说

依赖跟踪的初始描述掩盖了一些细节，这些细节在实际尝试实现时很快就变得令人头疼。特别是，我们很容易忽视我们正在处理的问题*二*依赖关系图：我们在前一个编译会话中构建的，以及我们为当前编译会话构建的。

当编译会话开始时，编译器将前一个依赖关系图作为不可变的数据块加载到内存中。然后，当调用查询时，它将首先尝试将图中的相应节点标记为绿色。这实际上意味着我们正在尝试在*以前的*DEP GRAPH为绿色，与中的查询键相对应。*现在的*会议。如何在当前查询键和上一个查询键之间进行映射`DepNode`是吗？答案是`Fingerprint`S：依赖关系图中的节点由查询键的指纹标识。由于指纹在编译会话中是稳定的，因此计算当前会话中的指纹可以让我们在上一个会话的依赖关系图中找到一个节点。如果我们找不到具有给定指纹的节点，这意味着查询键引用了前一个会话中尚未存在的内容。

因此，在上一个依赖关系图中找到DEP节点后，我们可以查找它的依赖关系（也可以查找上一个图中的DEP节点），然后继续使用Try-Mark-Green算法的其余部分。下一个有趣的事情发生在我们成功地将节点标记为绿色时。在这一点上，我们将节点和边复制到它的依赖关系中，从旧图复制到新图中。我们必须这样做，因为新的DEP图不能通过常规的依赖跟踪获取节点和边。跟踪系统只能在实际运行查询时记录边缘——但运行查询（尽管我们已经缓存了结果）正是我们想要避免的。

编译会话完成后，所有未更改的部分都已从旧的复制到新的依赖关系图中，而更改的部分已由跟踪系统添加到新的关系图中。此时，新的图将与查询结果缓存一起序列化到磁盘，并可以在随后的编译会话中充当前一个DEP图。

## 你忘了什么吗？：缓存升级

托多

# 未来：当前系统的缺陷和可能的解决方案

托多

[query-model]: ./query-evaluation-model-in-detail.html
