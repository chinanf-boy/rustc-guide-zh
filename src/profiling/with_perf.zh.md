# 性能分析

这是一个关于如何用[pref](https://perf.wiki.kernel.org/index.php/Main_Page)，分析 rustc 的指南。

## 初始步骤

- 获得一个干净的 rust-lang/master 分支，或任何你想分析的版本。
- 在您的`config.toml`：
  - `debuginfo-lines = true`
  - `use-jemalloc = false`-让你用 Valgrind 做内存分析
  - 把其他的都保留为默认值
- 运行`./x.py build`得到一个完整的构建版本
- 制作一个指向该结果的 Rust 工具链
  - 看见[有关说明，请参阅“构建和运行”章节。][b-a-r]

[b-a-r]: ../how-to-build-and-run.zh.html#toolchain

## 收集一个 pref profile

Perf 是 Linux 上一个优秀的工具，可以用来收集和分析各种信息。大多数情况下，它是用来计算程序在哪里花费时间的。但它也可以用于其他类型的事件，如缓存缺失等。

### 基础知识

基本`perf`命令如下：

```bash
> perf record -F99 --call-graph dwarf XXX
```

这个`-F99`告诉 Perf 以 99 Hz 采样，这样可以避免过长运行，而生成太多数据（为什么要求 99 Hz？通常选择的原因是，它不太可能与其他周期性活动同步）。这个`--call-graph dwarf`告诉 perf 从 debuginfo 获取调用图信息，精确的。这个`XXX`是要分析的命令。例如，您可以这样做：

```bash
> perf record -F99 --call-graph dwarf cargo +<toolchain> rustc
```

接着运行`cargo` —— 这里`<toolchain>`应该是您在开始时，创建的工具链名称。但是有一些事情需要注意：

- 您可能不想分析构建依赖，所花费的时间。所以像`cargo build; cargo clean -p $C`这样的命令，可能有所帮助（这里的`$C`是箱子名称）
  - 虽然，通常我只是`touch src/lib.rs`，然后重构建而已。=）
- 你可能不想让你的 profile 变得越来越乱。所以像`CARGO_INCREMENTAL=0`可能会有所帮助。

### 从`perf.rust-lang.org`测试，收集一个 pref profile

通常我们想把`perf.rust-lang.org`的某个测试，分析分析。为此，第一步是克隆[rustc-perf 存储库][rustc-perf-gh]：

```bash
> git clone https://github.com/rust-lang-nursery/rustc-perf
```

[rustc-perf-gh]: https://github.com/rust-lang-nursery/rustc-perf

#### 简单做

一旦克隆了 repo，就可以使用`collector`可执行文件为您进行分析！你可以找到[rustc-perf 的 readme 文件中的说明][rustc-perf-readme].

[rustc-perf-readme]: https://github.com/rust-lang-nursery/rustc-perf/blob/master/collector/README.md#profiling

例如，要测 clap-rs 测试，可以执行以下操作：

```bash
> ./target/release/collector
    --output-repo /path/to/place/output
    profile perf-record
    --rustc /path/to/rustc/executable/from/your/build/directory
    --cargo `which cargo`
    --filter clap-rs
    --builds Check
```

您也可以用相同的命令，来搭配 cachegrind 或其他分析工具。

#### 专挑难的做

如果您更喜欢手动运行，某些人会有这样的需求。首先需要找到所需测试的源代码。源测试位于[这个`collector/benchmarks`目录][dir]。那么，让我们进入一个特定测试的目录；例如，我们将使用`clap-rs`：

[dir]: https://github.com/rust-lang-nursery/rustc-perf/tree/master/collector/benchmarks

```bash
> cd collector/benchmarks/clap-rs
```

在这种情况下，假设我们想分析`cargo check`性能。在这种情况下，我将首先运行一些基本命令，来构建依赖项：

```bash
# 步骤: 清除旧项 和 构建依赖:
> cargo +<toolchain> clean
> CARGO_INCREMENTAL=0 cargo +<toolchain> check
```

（再次说明，`<toolchain>`应该替换成，我们在第一步中创建的工具链名称。）

下一步：我们*只是*要记录 clap-rs 箱子的执行时间，运行`cargo check`。我比较欢喜`cargo rustc`，因为它还允许我添加显式标志，这样我们就能稍后秀一波。

```bash
> touch src/lib.rs
> CARGO_INCREMENTAL=0 perf record -F99 --call-graph dwarf cargo rustc --profile check --lib
```

注意最终的命令：这是一个涂鸦！它使用`cargo rustc`命令，用（可能的）附加选项，来执行 rustc；而`--profile check`和`--lib`选项，一个是表明执行`cargo check`，和一个表明是一个 lib(库)（不是二进制）。

此时，我们就可以使用`perf`分析结果的工具。例如：

```bash
> perf report
```

将打开一个交互式 TUI 程序。在简单情况下，这是有帮助的。为了更详细的检查，[`perf-focus`工具][pf]可能会有所帮助；下面会介绍。

**注意事项。**每个 rustc-perf 测试都是自己的'电视雪花'。尤其啊，其中一些不是库，在这种情况下，您可能希望搞个`touch src/main.rs`，能避免传递`--lib`。我也不知道，怎样才能最好地分辨出，哪个测试是真实有效的。

### 正在收集 NLL 数据

如果你想分析一个 NLL 运行，你可以将额外的选项传递给`cargo rustc`命令，就像这样：

```bash
> touch src/lib.rs
> CARGO_INCREMENTAL=0 perf record -F99 --call-graph dwarf cargo rustc --profile check --lib -- -Zborrowck=mir
```

[pf]: https://github.com/nikomatsakis/perf-focus

## 使用`perf focus`分析 perf profile

一旦，你收集了一个 perf profile，我们会想得到一些关于它的信息。为此，我个人使用[perf focus][pf]。它是一种简单但有用的工具，可以帮您回答，以下问题：

- “函数 F 花费了多少时间”（无论从何处调用）
- “在 G 中调用函数 F 时，在函数 F 中花费了多少时间”
- “*排除*在 G 中花费的时间，在函数 F 中花费了多少时间”
- “F 调用什么函数，以及在这些函数中花费了多少时间”

要理解它是如何工作的，你必须对 perf 有一点了解。perf 基本的工作方式，是定期对您的流程*抽样*（或每当发生某些事件时）。对于每个样本，perf 收集一个回溯。`perf focus`允许您编写一个正则表达式，匹配测试出现在该回溯中的函数，然后告诉您哪些样本，具有符合该正则表达式的回溯。通过走一遍，我如何分析 NLL 性能的过程，可能让你最容易明白。

### 安装`perf-focus`

可以使用`cargo install`，安装 perf-focus：

```bash
> cargo install perf-focus
```

### 例子：在 MIR borrowck 中花了多少时间？

假设，我们已经收集到一个测试的 NLL 数据。我们想知道它在 MIR borrow-checker 花了多少时间。MIR borrowck 的“主”函数叫做`do_mir_borrowck`，因此我们可以执行以下命令：

```bash
> perf focus '{do_mir_borrowck}'
Matcher    : {do_mir_borrowck}
Matches    : 228
Not Matches: 542
Percentage : 29%
```

这个`'{do_mir_borrowck}'`参数被称为**匹配器**。它指定要应用测试的回溯。在这种情况下，`{X}`表示在回溯中必须存在，*一些*满足正则表达式的函数`X`。 在本例中，正则式只是我们想要的函数名称（但实际上，它是名称的一个子集；全名包括许多其他东西，比如模块路径，毕竟它是个正则式格式）。在这种模式下，Perf Focus 只打印出，`do_mir_borrowck`在栈中的百分比：在这个案例中，占 29%。

**关于 c++filt 的提醒。**要从`perf`中获取数据，`perf focus`当前会执行`perf script`（也许有更好的方法…）。我有时发现`perf script`输出 C++的杂乱名称。这很烦人。你可以通过运行`perf script | head`整顿整顿 —— 如果你看到名字是像`5rustc6middle`，而不是`rustc::middle`，那么你和我有同样的问题。您可以通过执行以下操作来解决此问题：

```bash
> perf script | c++filt | perf focus --from-stdin ...
```

通过管道传输，`perf script`的输出接上`c++filt`，这样应该将这些名称大概率转换成更友好的格式。这个`--from-stdin`标志传递给`perf focus`，是告诉它从 stdin 获取数据，而不是执行`perf focus`。 我们应该让这个更方便（最坏的情况下，可能会增加一个`c++filt`选项给`perf focus`，或者 像这样使用它-它很无害）。

### 例子：MIR borrowck 花了多少时间来搞定 traits 方面？

也许，我们想知道 MIR borrowck 在 traits 检测上花了多少时间。我们可以使用更复杂的正则式，问问：

```bash
> perf focus '{do_mir_borrowck}..{^rustc::traits}'
Matcher    : {do_mir_borrowck},..{^rustc::traits}
Matches    : 12
Not Matches: 1311
Percentage : 0%
```

这里我们用了`..`运算符，问“`do_mir_borrowck`在堆栈上，然之后，一些函数的名称以`rusc::traits`开始”（基本上，在该模块中编码）。结果是答案是“几乎从来没有”——只有 12 个样本符合这个描述（如果你每每都看到*没*示例，这通常表示您搞乱了查询）。

如果您好奇，可以通过使用`--print-match`选项，查看详情。这将打印出每个样本的完整回溯。这个`|`行的前面表示正则表达式匹配的部分。

### 例子：MIR borrowck 在哪里消磨时间？

通常，我们想做一个更“探索性”的查询。比如，我们知道 MIR borrowck 有 29% 的时间占比，但是这段时间是在哪里度过的呢？不知道吧。为此，`--tree-callees`选择往往是最好的工具。你也能给个`--tree-min-percent`或`--tree-max-depth`选项，结果如下：

```bash
> perf focus '{do_mir_borrowck}' --tree-callees --tree-min-percent 3
Matcher    : {do_mir_borrowck}
Matches    : 577
Not Matches: 746
Percentage : 43%

Tree
| matched `{do_mir_borrowck}` (43% total, 0% self)
: | rustc_mir::borrow_check::nll::compute_regions (20% total, 0% self)
: : | rustc_mir::borrow_check::nll::type_check::type_check_internal (13% total, 0% self)
: : : | core::ops::function::FnOnce::call_once (5% total, 0% self)
: : : : | rustc_mir::borrow_check::nll::type_check::liveness::generate (5% total, 3% self)
: : : | <rustc_mir::borrow_check::nll::type_check::TypeVerifier<'a, 'b, 'gcx, 'tcx> as rustc::mir::visit::Visitor<'tcx>>::visit_mir (3% total, 0% self)
: | rustc::mir::visit::Visitor::visit_mir (8% total, 6% self)
: | <rustc_mir::borrow_check::MirBorrowckCtxt<'cx, 'gcx, 'tcx> as rustc_mir::dataflow::DataflowResultsConsumer<'cx, 'tcx>>::visit_statement_entry (5% total, 0% self)
: | rustc_mir::dataflow::do_dataflow (3% total, 0% self)
```

用`--tree-callees`所会发生的，就是：

- 我们发现每个样本，都匹配正则表达式
- 在正则式匹配*之后*，工具看看发现的代码，并尝试建立调用树

这个`--tree-min-percent 3`选项说“只显示超过 3%的时间。如果没有这个，树通常会变得非常嘈杂，包括一些随机的东西，比如 malloc 的内部。`--tree-max-depth`也很有用，但它只是限制了我们打印的级别。

对于每一行，我们将显示总时间下，该函数（"total 全部"）所占的百分比，和自个函数所占百分比。**只是那个函数，而不是那个函数的某个下层调用**（就是 self{自个}）。通常“total”是个更有趣，和代表性的数字，但并非总是如此。

### 相对百分比

默认情况下，perf-focus 的所有计算，都是相对于**程序执行总时间**。 这有助于您保持观察全局 —— 但通常当我们深入研究，寻找热点时，我们会忽略一个事实，即在整个程序执行中，这个“热点”实际上并不重要。而相对性不同，它能确保不同查询之间的百分比，进行容易的比较。

也就是说，有时候，相对百分比是有用的，所以`perf focus`提供一个`--relative`选项。在这种情况下，仅列出匹配样本的百分比（仅与所有样本相比）。例如，我们可以得到相对于，borrowck 本身的百分比，就像这样：

```bash
> perf focus '{do_mir_borrowck}' --tree-callees --relative --tree-max-depth 1 --tree-min-percent 5
Matcher    : {do_mir_borrowck}
Matches    : 577
Not Matches: 746
Percentage : 100%

Tree
| matched `{do_mir_borrowck}` (100% total, 0% self)
: | rustc_mir::borrow_check::nll::compute_regions (47% total, 0% self) [...]
: | rustc::mir::visit::Visitor::visit_mir (19% total, 15% self) [...]
: | <rustc_mir::borrow_check::MirBorrowckCtxt<'cx, 'gcx, 'tcx> as rustc_mir::dataflow::DataflowResultsConsumer<'cx, 'tcx>>::visit_statement_entry (13% total, 0% self) [...]
: | rustc_mir::dataflow::do_dataflow (8% total, 1% self) [...]
```

你可以看到，`compute_regions`占了 47%——这意味着`do_mir_borrowck`的时间是花在那个函数上的。以前，我们看到 20% —— 那是因为`do_mir_borrowck`其本身，仅占总时间的 43%（以及`.47 * .43 = .20`，说明单`compute_regions`就占总时间的 20%）
