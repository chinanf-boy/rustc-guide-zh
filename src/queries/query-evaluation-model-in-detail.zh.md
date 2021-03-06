# 详细的查询执行模型

本章是关于构建的查询，并深入研究其抽象模型。它不涉及实现细节，而是试图解释底层逻辑。因此，这里的示例已经被简化，并没有直接反映编译器的内部 API。

## 什么是查询？

抽象地说，我们把编译器对给定箱子的认识，视为一个“数据库”，而查询是询问编译器关于该箱子问题的方式，也就是说，我们“查询”编译器的“数据库”，以了解事实。

但是，这个编译器数据库有一些特殊之处：它从无开始，在执行查询时按需填充。因此，如果数据库尚未包含一个查询的结果，那么这个查询必须知道如何计算其结果。为此，它可以访问其他查询，和数据库在创建时，预先填充的某些输入值。

因此，查询由以下内容组成：

- 标识一个查询的名称
- 指定要查找内容的“键(key)”
- 指定回馈哪种结果的结果类型
- 一个“provider”，它是一个函数，在数据库没有结果的情况下，指示如何计算结果。

例如，一个`type_of`查询的名称，就是`type_of`，它的查询键是一个`DefId`，用来表示我们想要知道关于其类型的项，结果类型是`Ty<'tcx>`，还有一个 provider 函数，获取查询键和对数据库其余部分的访问权限，可以通过查询键，计算标识项的类型。

所以在某种意义上，查询只是一个将查询键，映射到相应结果的函数。但是，我们必须应用一些限制，使其听起来合理：

- 键和结果必须是不可变的值。
- provider 函数必须是纯函数，也就是说，对于同一个键，它必须始终产生相同的结果。
- provider 函数唯一所采用的参数，是键和对“查询上下文”的引用（它提供对“数据库”其余部分的访问）。

通过调用查询，可以轻松地建立数据库。查询的 provider 函数将调用其他查询，这些查询的结果已被缓存，或通过调用其他查询的 provider 函数 计算出来。这些查询 provider 函数的调用，在概念上形成了一个有向无环图（DAG），其中一块'叶子'是在创建查询上下文时，已经知道的输入值。

## 缓存化/记忆化

查询调用的结果是“memoized{记住的}”，这意味着查询上下文将结果缓存在内部表格中，当再次使用相同的查询键，调用该查询时，将返回缓存的结果，而不是再次运行 provider。

这种缓存对于提高查询引擎的效率至关重要。如果没有记忆化，系统仍然是健全的（也就是说，它将产生相同的结果），但会重复相同的计算。

记忆化 是查询的 provider 函数 必须是纯函数的主要原因之一。如果调用一个 provider 函数，但每次调用都产生不同的结果（因为它访问一些全局可变状态），那么我们就不能把上一结果记住并拿来用。

## 输入数据

当创建查询上下文时，它仍然为空：未执行任何查询，未缓存任何结果。但是上下文已经提供了对“输入”数据的访问，即在创建上下文之前，计算出来的不可变数据块，并且查询可以访问这些数据，来进行计算。目前，这个输入数据主要由 HIR map ，和编译器调用时使用的命令行选项组成。将来，输入将只包含命令行选项和源文件列表 —— HIR map 本身将，由一个处理这些源文件的查询提供。

如果没有输入，查询将处于无效状态，因没有任何内容去计算结果（请记住，查询 provider 只能访问其他查询和上下文，而不能访问任何其他外部状态或信息）。

对于一个查询的 provider 函数，输入数据和其他查询的结果，看起来完全相同：它只告诉上下文，“给我 x 的值”。因为输入数据是不可变的，所以 provider 可以依赖它的不可变性，在不同的查询中调用，就像查询结果一样。

## 跟踪一些查询的执行示例

这个 查询调用的 DAG(有向无环图)是如何产生的？在某种程度上，编译器驱动程序将创建查询上下文（目前还为空）。然后，它将从查询系统外部，调用查询所需要的，再去执行其任务。如下所示：

```rust,ignore
fn compile_crate() {}
    let cli_options = ...;
    let hir_map = ...;

    // 创建 query context(查询上下文) `tcx`
    let tcx = TyCtxt::new(cli_options, hir_map);

    // 通过调用类型测试查询，去做类型测试
    tcx.type_check_crate();
}
```

这个`type_check_crate`查询的 provider 函数， 如下所示：

```rust,ignore
fn type_check_crate_provider(tcx, _key: ()) {
    let list_of_items = tcx.hir_map.list_of_items();

    for item_def_id in list_of_hir_items {
        tcx.type_check_item(item_def_id);
    }
}
```

我们看到了`type_check_crate`查询访问输入数据（`tcx.hir_map.list_of_items()`），并调用其他查询（`type_check_item`）。这个`type_check_item`查询，或许自己也访问输入数据和/或调用其他的查询，所以到最后，查询的调用 DAG (图)，会从起初执行的节点开始，往后构建成形。

```ignore
         (2)                                                 (1)
  list_of_all_hir_items <----------------------------- type_check_crate()
                                                               |
    (5)             (4)                  (3)                   |
  Hir(foo) <--- type_of(foo) <--- type_check_item(foo) <-------+
                                      |                        |
                    +-----------------+                        |
                    |                                          |
    (7)             v  (6)                  (8)                |
  Hir(bar) <--- type_of(bar) <--- type_check_item(bar) <-------+

// (x) denotes invocation order
```

我们也可以看到，一个查询结果，往往可以从缓存中读取：`type_of(bar)`计算结果，是`type_check_item(foo)`的。所以当`type_check_item(bar)`需要它，它就在缓存里。

查询结果缓存会一直存在，只要其查询上下文还'活着'。所以如果编译驱动程序之后调用另一个查询，那么上图会一直存在，和所有已执行的查询(结果)不需要再做。

## 周期

前面，我们用一个 DAG，描述了查询调用的状况。然而，这很容易形成一个循环图，例如，有一个像下面的查询 provider：

```rust,ignore
fn cyclic_query_provider(tcx, key) -> u32 {
  // 用同个key，再次调用同个查询
  tcx.cyclic_query(key)
}
```

由于查询 provider 是正则式函数，预期的行为表现为：执行会停滞在一个无限递归中。一个像这样的查询，是不会有太大用处的。然而，有时候在查询中，为了确认某些用户无效输入，可用一种循环方式来调用查询结果。查询引擎包括一个对循环调用的检查，因为循环是个没救的错误，后果就是，中止执行，并给出一个人性化的"cycle error"信息。

到了编译器的某个点上，产生了一个想法：“回收循环”，这是一个可以“尝试”执行一个查询，如果它最后会导致一个循环，就分流到其他方式。然而，这个想法稍后删除了，因为它不确定最终会导致什么样的后果，尤其是对增量编译的影响。

## “窃取”查询

有些查询的结果包装在`Steal<T>`结构。这些查询的行为与常规查询完全相同，但有一个例外：它们的结果在某个时刻可能会被“窃取”出缓存，这意味着程序的其他部分将拥有它的所有权，并且无法再访问结果。

这种窃取机制纯粹是作为性能优化而存在的，因为有些结果值的克隆成本太高（例如一个函数的 MIR）。似乎看起来，窃取结果会违反，查询结果必须是不可变的条件（毕竟我们正在将结果值移出缓存），但只要可变操作不可见，就可以。这是通过两件事实现的：

- 在结果被窃取之前，我们确保急切运行，可能要读取该结果的所有查询。这必须手动调用这些查询，来完成。
- 每当一个查询试图访问一个已被盗的结果时，我们会使编译器结冰，这样，一个条件(问题)都不会错过。

这不是一个理想的设置，因为需要手动完成，所以应该谨慎使用，并且只有当，有所了解哪些查询可以访问给定的结果时，才使用它。然而，在实践中，窃取并没有一个大的维护负担。

总而言之：“窃取查询”以可控的方式破坏了一些规则。也存在一些检查，以确保没有任何事情可以悄悄地出错。

## 并行查询执行

查询模型具有一些特性，让多个查询并行执行，实际上是可行的，而无需付出太多的努力：

- 查询 provider 可以通过查询上下文访问，访问的所有数据，因此查询上下文可以负责同步访问。
- 查询结果必须是不可变的，这样，不同的线程就可以安全地同时使用它们。

夜间编译器已经实现了，如下并行查询执行：

当一个`foo`查询执行时，`foo`的缓存表被锁定。

- 如果已经有了结果，我们可以克隆它，释放锁，然后就完成了。
- 如果没有缓存条目，也没有其他计算同个结果的活跃查询调用，那么我们将该键标记为“正在进行(in progress)”，释放锁并开始计算。
- 如果*有*对同一个键的另一个，正在进行的查询调用，那我们释放**那个**锁，然后阻塞自己线程，直到另一个调用计算出我们等待的结果为止。这不能死锁，因为正如前面提到的，查询调用形成一个 DAG。最终，线程总能前进。
