# 调试和测试依赖项

## 测试依赖关系图

有多种方法可以根据依赖关系图编写测试。最简单的机制是`#[rustc_if_this_changed]`和`#[rustc_then_this_would_need]`注释。这些用于编译失败测试，以测试依赖关系图中是否存在预期的路径集。例如，请参见`src/test/compile-fail/dep-graph-caller-callee.rs`.

其思想是您可以对测试进行如下注释：

```rust,ignore
#[rustc_if_this_changed]
fn foo() { }

#[rustc_then_this_would_need(TypeckTables)] //~ ERROR OK
fn bar() { foo(); }

#[rustc_then_this_would_need(TypeckTables)] //~ ERROR no path
fn baz() { }
```

这将检查依赖关系图中是否有来自的路径`Hir(foo)`到`TypeckTables(bar)`. 每个都会报告一个错误`#[rustc_then_this_would_need]`指示路径是否存在的批注。`//~ ERROR`然后，可以使用注释来测试是否找到路径（如上所示）。

## 调试依赖关系图

### 转储图形

编译器还可以为您的调试乐趣转储依赖关系图。为此，通过`-Z dump-dep-graph`旗帜。图形将被转储到`dep_graph.{txt,dot}`在当前目录中。您可以使用`RUST_DEP_GRAPH`环境变量。

然而，通常情况下，完整的DEP图是非常令人难以抗拒的，并没有特别的帮助。因此，编译器还允许您过滤图形。您可以通过三种方式进行过滤：

1.  源于一组特定节点（通常是单个节点）的所有边。
2.  到达特定节点集的所有边。
3.  位于给定开始节点和结束节点之间的所有边。

要筛选，请使用`RUST_DEP_GRAPH_FILTER`环境变量，应类似于以下内容之一：

```text
source_filter     // nodes originating from source_filter
-> target_filter  // nodes that can reach target_filter
source_filter -> target_filter // nodes in between source_filter and target_filter
```

`source_filter`和`target_filter`是一个`&`-字符串的分隔列表。如果所有这些字符串都出现在节点的标签中，则该节点被视为匹配筛选器。例如：

```text
RUST_DEP_GRAPH_FILTER='-> TypeckTables'
```

将选择所有`TypeckTables`节点。通常尽管你想要`TypeckTables`节点用于某些特定的fn，因此您可以编写：

```text
RUST_DEP_GRAPH_FILTER='-> TypeckTables & bar'
```

这将仅选择的前置任务`TypeckTables`具有的函数的节点`bar`以他们的名义。

也许当你改变的时候你会发现`foo`你需要重新打字检查`bar`但是你不认为你应该这样做。在这种情况下，您可以这样做：

```text
RUST_DEP_GRAPH_FILTER='Hir & foo -> TypeckTables & bar'
```

这将转储所有从`Hir(foo)`到`TypeckTables(bar)`从中（希望）可以看到错误边缘的来源。

### 跟踪不正确的边缘

有时，在转储依赖关系图之后，您会发现一些不应该存在的路径，但您不太确定它是如何形成的。**当使用调试断言构建编译器时，**它可以帮助你找到答案。简单设置`RUST_FORBID_DEP_GRAPH_EDGE`环境变量到筛选器。在DEP图中创建的每个边都将根据该过滤器进行测试-如果匹配，则`bug!`是报告的，因此您可以很容易地看到回溯（`RUST_BACKTRACE=1`）。

这些过滤器的语法与前一节中描述的相同。但是，请注意，此筛选器应用于**边缘**与前一节不同，它不会处理图形中较长的路径。

例子：

你发现有一条路从`Hir`属于`foo`到的类型检查`bar`你不认为应该有。如前一节所述，转储DEP图并打开`dep-graph.txt`看到类似的东西：

```text
Hir(foo) -> Collect(bar)
Collect(bar) -> TypeckTables(bar)
```

你觉得第一个边缘很可疑。所以你定了`RUST_FORBID_DEP_GRAPH_EDGE`到`Hir&foo -> Collect&bar`，重新运行，然后观察回溯。喂，虫子修好了！
