# 调试和测试依赖项

## 测试依赖关系图

有多种方法，可根据依赖关系图编写测试。最简单的机制是`#[rustc_if_this_changed]`和`#[rustc_then_this_would_need]`注释。这些用于得到该值测试，以测试依赖关系图中是否存在预期的路径集。例如，请参见`src/test/compile-fail/dep-graph-caller-callee.rs`.

想法是您可以对测试进行如下注释：

```rust,ignore
#[rustc_if_this_changed]
fn foo() { }

#[rustc_then_this_would_need(TypeckTables)] //~ ERROR OK
fn bar() { foo(); }

#[rustc_then_this_would_need(TypeckTables)] //~ ERROR no path
fn baz() { }
```

这将检查依赖关系图中是否有，从`Hir(foo)`到`TypeckTables(bar)`的路径。 指示路径是否存在`#[rustc_then_this_would_need]`批注，它们每个都会报告一个错误。然后，可以使用`//~ ERROR`注释来测试是否找到路径（如上所示）。

## 调试依赖关系图

### 转存图形

编译器还乐意帮您的调试，转存依赖关系图。为此，通过`-Z dump-dep-graph`标志。图形将被转存到，当前目录中`dep_graph.{txt,dot}`。您可以使用`RUST_DEP_GRAPH`环境变量，覆盖文件名。

然而，通常情况下，完整的 de 图是非常难受香菇的，并没有很大帮助。因此，编译器还允许您过滤图形。三种方式进行过滤：

1.  源于一组特定节点的所有边（常用于，单个节点）。
2.  到达特定节点集的所有边。
3.  给定开始节点和结束节点，它们之间的所有边。

要筛选，请使用`RUST_DEP_GRAPH_FILTER`环境变量，应类似于以下内容之一：

```text
source_filter     // 源于 source_filter 的节点
-> target_filter  // 可以到达 target_filter 的节点
source_filter -> target_filter // source_filter 和 target_filter之间的节点
```

`source_filter`和`target_filter`是一个`&`-字符串的分隔列表。如果所有这些字符串，都出现在节点的标签中，则该节点要去匹配一个过滤器。例如：

```text
RUST_DEP_GRAPH_FILTER='-> TypeckTables'
```

会选择所有`TypeckTables`节点的前节点。常常你想要`TypeckTables`节点，是跟着某些特定的`fn`，因此您可以编写：

```text
RUST_DEP_GRAPH_FILTER='-> TypeckTables & bar'
```

会选择所有`TypeckTables`节点的前节点，且他们名字，具有`bar`函数。

也许，当你改变`foo`的时候，会发现你需要重新类型检查下`bar`，但是你不认为你应该这样做。在这种情况下，您可以这样做：

```text
RUST_DEP_GRAPH_FILTER='Hir & foo -> TypeckTables & bar'
```

这将转存所有从`Hir(foo)`到`TypeckTables(bar)`的节点，从中（希望）可以看到错误边的源。

### 跟踪不正确的边

有时，在转存依赖关系图之后，您会发现一些不应该存在的路径，但您不太确定它是如何形成的。**当使用调试断言功能，构建编译器时，**它可以帮助你找到答案。简单设置`RUST_FORBID_DEP_GRAPH_EDGE`环境变量到一个过滤器。在 dep-graph 中创建的每个边，都将根据该过滤器进行测试—— 如果匹配，则`bug!`会报告，因此您可以很容易地看到回溯（`RUST_BACKTRACE=1`）。

这些过滤器的语法与前一节中描述的相同。但是，请注意，此过滤器应用于每条**边**，且与前一节不同，它不会处理图形中较长的路径。

例子：

你发现有一条路，其从`foo`的`Hir`（`Hir(foo)`），到`bar`的类型检查（`TypeckTables(bar)`），和你不认为应该有这条路。如前一节所述，转存 dep-graph，并打开`dep-graph.txt`，你会看到类似的东西：

```text
Hir(foo) -> Collect(bar)
Collect(bar) -> TypeckTables(bar)
```

你觉得第一条边很可疑。所以你把`RUST_FORBID_DEP_GRAPH_EDGE`设为`Hir&foo -> Collect&bar`，重新运行，然后观察回溯。瞧，bug 修好了！
