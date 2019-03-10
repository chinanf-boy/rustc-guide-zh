# 性能分析

这是一个关于如何用[pref](https://perf.wiki.kernel.org/index.php/Main_Page)，分析 rustc 的指南。

## 初始步骤

- 获得一个干净的 rust-lang/master 分支，或任何你想分析的版本。
- 在您的`config.toml`：
  - `debuginfo-lines = true`
  - `use-jemalloc = false`-让你用 Valgrind 做内存分析
  - 把其他的都保留为默认值
- 跑`./x.py build`得到一个完整的版本
- 制作一个指向该结果的生锈工具链
  - 看见[有关说明，请参阅“构建和运行”部分。][b-a-r]

[b-a-r]: ../how-to-build-and-run.html#toolchain

## 收集性能配置文件

Perf 是 Linux 上一个优秀的工具，可以用来收集和分析各种信息。大多数情况下，它是用来计算程序在哪里花费时间的。但它也可以用于其他类型的事件，如缓存未命中等。

### 基础知识

基本`perf`命令如下：

```bash
> perf record -F99 --call-graph dwarf XXX
```

这个`-F99`告诉 Perf 以 99 赫兹采样，这样可以避免为长时间运行生成太多数据（为什么要求 99 赫兹？通常选择它是因为它不太可能与其他周期性活动保持同步）。这个`--call-graph dwarf`告诉 perf 从 debuginfo 获取调用图信息，这是准确的。这个`XXX`是要分析的命令。例如，您可以这样做：

```bash
> perf record -F99 --call-graph dwarf cargo +<toolchain> rustc
```

运行`cargo`——这里`<toolchain>`应该是您在开始时创建的工具链的名称。但是有一些事情需要注意：

- 您可能不想分析构建依赖关系所花费的时间。所以有点像`cargo build; cargo clean -p $C`可能有帮助（在哪里`$C`是板条箱的名称）
  - 虽然通常我只是`touch src/lib.rs`改为重建。=）
- 你可能不想让你的个人资料变得越来越乱。所以有点像`CARGO_INCREMENTAL=0`可能会有所帮助。

### 从中收集性能配置文件`perf.rust-lang.org`测试

通常我们想从`perf.rust-lang.org`.为此，第一步是克隆[Rustc 性能存储库][rustc-perf-gh]以下内容：

```bash
> git clone https://github.com/rust-lang-nursery/rustc-perf
```

[rustc-perf-gh]: https://github.com/rust-lang-nursery/rustc-perf

#### 用简单的方法

一旦克隆了 repo，就可以使用`collector`可执行文件为您进行分析！你可以找到[Rustc Perf 自述文件中的说明][rustc-perf-readme].

[rustc-perf-readme]: https://github.com/rust-lang-nursery/rustc-perf/blob/master/collector/README.md#profiling

例如，要测量拍手器测试，可以执行以下操作：

```bash
> ./target/release/collector
    --output-repo /path/to/place/output
    profile perf-record
    --rustc /path/to/rustc/executable/from/your/build/directory
    --cargo `which cargo`
    --filter clap-rs
    --builds Check
```

您也可以使用相同的命令来使用 cachegrind 或其他分析工具。

#### 努力做这件事

如果您喜欢手动运行，这也是可能的。首先需要找到所需测试的源。测试源位于[这个`collector/benchmarks`目录][dir].那么，让我们进入一个特定测试的目录；我们将使用`clap-rs`例如：

[dir]: https://github.com/rust-lang-nursery/rustc-perf/tree/master/collector/benchmarks

```bash
> cd collector/benchmarks/clap-rs
```

在这种情况下，假设我们想分析`cargo check`性能。在这种情况下，我将首先运行一些基本命令来构建依赖项：

```bash
# Setup: first clean out any old results and build the dependencies:
> cargo +<toolchain> clean
> CARGO_INCREMENTAL=0 cargo +<toolchain> check
```

（再一次，`<toolchain>`应该用我们在第一步中创建的工具链的名称替换。）

下一步：我们要记录*只是*拍板板条箱，检查货物。我倾向于使用`cargo rustc`为此，因为它还允许我添加显式标志，稍后我们将进行此操作。

```bash
> touch src/lib.rs
> CARGO_INCREMENTAL=0 perf record -F99 --call-graph dwarf cargo rustc --profile check --lib
```

注意最后的命令：这是一个涂鸦！它使用`cargo rustc`命令，使用（可能的）附加选项执行 rustc；命令`--profile check`和`--lib`选项指定我们正在执行`cargo check`执行，这是一个库（不是二进制）。

此时，我们可以使用`perf`分析结果的工具。例如：

```bash
> perf report
```

将打开一个交互式 TUI 程序。在简单的情况下，这是有帮助的。为了更详细的检查，[`perf-focus`工具][pf]可能会有所帮助；下面介绍。

**注意事项。**每个 Rustc 性能测试都是它自己的特殊雪花。尤其是，其中一些不是库，在这种情况下，您可能希望这样做`touch src/main.rs`避免超车`--lib`.我不知道怎样才能最好地分辨出哪个测试是诚实的。

### 正在收集 NLL 数据

如果你想分析一个 NLL 运行，你可以将额外的选项传递给`cargo rustc`命令，就像这样：

```bash
> touch src/lib.rs
> CARGO_INCREMENTAL=0 perf record -F99 --call-graph dwarf cargo rustc --profile check --lib -- -Zborrowck=mir
```

[pf]: https://github.com/nikomatsakis/perf-focus

## 使用分析性能配置文件`perf focus`

一旦你收集了一个性能配置文件，我们想得到一些关于它的信息。为此，我个人使用[性能焦点][pf].它是一种简单但有用的工具，可以让您回答以下问题：

- “函数 f 花费了多少时间”（无论从何处调用）
- “从 g 调用函数 f 时，在函数 f 中花费了多少时间”
- “在函数 f 中花费了多少时间*排除*在 G”中花费的时间
- “F 调用什么函数以及在函数中花费了多少时间”

要理解它是如何工作的，你必须对性能有一点了解。基本上，性能是由*抽样*您的流程是定期的（或每当发生某些事件时）。对于每个样本，perf 收集一个回溯。`perf focus`允许您编写一个正则表达式，该表达式测试出现在该回溯中的函数，然后告诉您哪些样本具有符合该正则表达式的回溯。通过浏览我将如何分析 NLL 性能，这可能是最容易解释的。

### 安装`perf-focus`

可以使用安装性能焦点`cargo install`：

```bash
> cargo install perf-focus
```

### 例子：在米尔·博洛克花了多少时间？

假设我们已经为测试收集了 NLL 数据。我们想知道它花了多少时间在和平号检查借阅。mir borrowck 的“main”函数被调用`do_mir_borrowck`，因此我们可以执行以下命令：

```bash
> perf focus '{do_mir_borrowck}'
Matcher    : {do_mir_borrowck}
Matches    : 228
Not Matches: 542
Percentage : 29%
```

这个`'{do_mir_borrowck}'`参数被称为**匹配器**. 它指定要应用于回溯的测试。在这种情况下，`{X}`表示必须存在*一些*满足正则表达式的回溯函数`X`. 在本例中，regex 只是我们想要的函数的名称（实际上，它是名称的一个子集；全名包括许多其他东西，比如模块路径）。在这种模式下，Perf Focus 只打印出样本的百分比，其中`do_mir_borrowck`在这个案例中，占 29%。

**关于 C++ FILT 的注释。**从中获取数据`perf`，`perf focus`当前执行`perf script`（也许有更好的方法…）。我有时发现`perf script`输出 C++的名称。这很烦人。你可以通过跑步来判断`perf script | head`你自己-如果你看到名字像`5rustc6middle`而不是`rustc::middle`，那么你也有同样的问题。您可以通过执行以下操作来解决此问题：

```bash
> perf script | c++filt | perf focus --from-stdin ...
```

这将通过管道传输`perf script`通过`c++filt`并且应该主要将这些名称转换成更友好的格式。这个`--from-stdin`旗到`perf focus`告诉它从 stdin 获取数据，而不是执行`perf focus`. 我们应该让这个更方便（最坏的情况下，可能会增加一个`c++filt`选择权`perf focus`或者总是使用它-它很无害）。

### MirBorrowck 花了多少时间来解决特征问题？

也许我们想知道 MirBorrowck 在特征检测上花了多少时间。我们可以使用更复杂的 regex 来问这个问题：

```bash
> perf focus '{do_mir_borrowck}..{^rustc::traits}'
Matcher    : {do_mir_borrowck},..{^rustc::traits}
Matches    : 12
Not Matches: 1311
Percentage : 0%
```

这里我们用了`..`接线员问“我们有多长时间`do_mir_borrowck`在堆栈上，然后，在后面，一些函数的名称以`rusc::traits`“（基本上，在该模块中编码）。结果是答案是“几乎从来没有”—只有 12 个样本符合这个描述（如果你曾经看到*不*示例，这通常表示您的查询混乱）。

如果您好奇，可以通过使用`--print-match`选择权。这将打印出每个样本的完整回溯。这个`|`行的前面表示正则表达式匹配的部分。

### 米尔博罗克在哪里消磨时间？

通常我们想做一个更“探索性”的查询。比如，我们知道米尔·博罗克有 29%的时间，但是这段时间是在哪里度过的呢？为此，`--tree-callees`选择往往是最好的工具。你通常也想给`--tree-min-percent`或`--tree-max-depth`. 结果如下：

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

发生了什么事`--tree-callees`那是

- 我们发现每个样本都匹配正则表达式
- 我们看看发生的代码*之后*regex 匹配并尝试建立调用树

这个`--tree-min-percent 3`选项说“只显示超过 3%的时间。如果没有这个，树通常会变得非常嘈杂，包括一些随机的东西，比如 malloc 的内部。`--tree-max-depth`也很有用，它只是限制了我们打印的级别。

对于每一行，我们将显示该函数中总时间百分比（“总计”）和所用时间百分比。**只是那个函数，而不是那个函数的某个被调用方**（自我）。通常“总数”是更有趣的数字，但并非总是如此。

### 相对百分比

默认情况下，所有性能焦点都是相对于**程序执行总数**. 这有助于您保持洞察力——通常当我们深入研究寻找热点时，我们会忽略一个事实，即在整个程序执行中，这个“热点”实际上并不重要。它还确保不同查询之间的百分比很容易进行比较。

也就是说，有时候得到相对百分比是有用的，所以`perf focus`提供一个`--relative`选择权。在这种情况下，百分比仅列出匹配的样本（与所有样本相比）。例如，我们可以得到相对于出生地本身的百分比，就像这样：

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

给你看`compute_regions`总计 47%——这意味着`do_mir_borrowck`是花在那个功能上的。以前，我们看到 20%——那是因为`do_mir_borrowck`其本身仅占总时间的 43%（以及`.47 * .43 = .20`）
