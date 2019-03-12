# 附录A：有关为rustc创建插件替换的教程

> **注意：**这是副本`@nrc`很棒[愚蠢的，统计-]。您应该在GitHub存储库中找到代码的副本，尽管由于编译器不断发展，但不能保证它会在第一时间编译。

许多工具都可以从编译器的替代品中受益。通过这个，我的意思是该工具的任何用户都可以使用`mytool`以他们通常使用的所有方式`rustc`- 无论是手动编译单个文件还是作为复杂的make项目或货物构建等的一部分，这可能是很多工作;与大多数编译器一样，rustc需要大量的命令行参数，这些参数会以复杂和交互的方式影响编译。在您的工具中模拟所有这些行为是令人讨厌的，特别是如果您在librustc中进行许多与编译器相同的调用。

我想到的东西是像rustdoc或未来的rustfmt这样的工具。这些想要尽可能接近真正的编译，但具有完全不同的输出（分别是文档和格式化的源代码）。另一个用例是自定义编译器。假设您想在宏扩展后添加自定义代码生成阶段，那么创建新工具应该比分配编译器更容易（并且随着编译器的发展使其保持最新）。

我已逐渐尝试改进librustc的API，以便更容易创建一个插入式工具（许多其他人也在同一时间框架内帮助改进了这些接口）。现在制作一个尽可能接近rustc的工具非常简单。在本教程中，我将展示如何。

注意/警告，我在本教程中讨论的所有内容都是rustc的内部API。它非常不稳定，可能经常以不可预测的方式改变。维护使用这些API的工具将是非常重要的，尽管希望比在不使用它们的情况下维护类似的东西更容易。

本教程从rustc编译过程和驱动编译的一些代码的高级视图开始。然后我将描述如何定制该过程。在本教程的最后一节中，我将通过一个示例--stupid-stats  - 展示如何构建一个插入式工具。

## 编译过程概述

使用rustc进行编译分几个阶段进行。我们从解析开始，这包括lexing。此阶段的输出是AST（抽象语法树）。每个包都有一个AST（实际上，整个编译过程在一个包中运行）。解析抽象出有关单个文件的详细信息，这些文件将在此阶段读入AST。在这个阶段，AST包括所有的宏用途，属性仍然存在，并且没有任何东西会被淘汰`cfg`秒。

下一阶段是配置和宏扩展。这可以被认为是AST的一个功能。未扩展的AST进入并且扩展的AST出来了。扩展了宏和语法扩展，并且`cfg`属性会导致某些代码消失。生成的AST将不会包含任何宏或宏用途。

前两个阶段的代码是[libsyntax](https://github.com/rust-lang/rust/tree/master/src/libsyntax)。

在此阶段之后，编译器将id分配给AST中的每个节点（技术上不是每个节点，而是大多数节点）。如果我们写出依赖关系，那么现在就会发生。

下一个重要阶段是分析。这是最复杂的阶段，并使用rustc中的大部分代码。这包括名称解析，类型检查，借用检查，类型和生命周期推断，特征选择，方法选择，linting等。大多数错误检测都是在此阶段完成的（尽管在解析期间发现了解析错误）。此阶段的“输出”是一组包含有关源程序的语义信息的边表。分析代码在[librustc](https://github.com/rust-lang/rust/tree/master/src/librustc)还有一堆带有'librustc\_'前缀的其他箱子。

接下来是转换，它将AST（以及所有这些副表）转换为LLVM IR（中间表示）。我们通过调用LLVM库来实现这一点，而不是直接将IR写入文件。此代码在librustc_trans中。

下一阶段是运行llvm后端。这将在生成的IR上运行LLVM的优化传递，然后生成机器代码。结果是对象文件。这个阶段全部由LLVM完成，它不是Rust编译器的一部分。llvm和rustc之间的接口在[藏书楼](https://github.com/rust-lang/rust/tree/master/src/librustc_llvm).

最后，我们将对象文件链接到一个可执行文件中。我们再次将它外包给其他程序，但它并不是Rust编译器的一部分。接口位于librustc_back中（它还包含一些主要在翻译过程中使用的东西）。

> 注：`librustc_trans`和`librustc_back`不再存在，我们不再将ast或hir直接转换为llvm-ir。相反，看到[`librustc_codegen_llvm`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_llvm/index.html)和[`librustc_codegen_utils`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_utils/index.html).

所有这些相位都由驾驶员协调。要查看确切的顺序，请查看[这个`compile_input`功能在`librustc_driver`][compile-input]. 驱动程序处理编译-1的所有最高级别的协调。处理命令行参数2。维护编译状态（主要在`Session`3）。调用适当的代码以运行编译4的每个阶段。处理漂亮打印和测试的高级协调。为了创建一个嵌入式编译器替换或编译器替换，我们将大部分编译单独进行，并使用其API自定义驱动程序。

[compile-input]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/driver/fn.compile_input.html

## 驱动程序自定义API

有两种主要的自定义编译方法-使用`CompilerCalls`并使用`CompileController`. 前者允许您自定义命令行参数等的处理，后者允许您提前停止编译或在阶段之间执行代码。

### `CompilerCalls`

`CompilerCalls`是您在工具中实现的特性。它包含一组非常特别的方法，可以挂接到处理命令行参数和驱动编译器的过程中。有关详细信息，请参见[librust驱动程序/lib.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/index.html). 我将在这里总结这些方法。

`early_callback`和`late_callback`让您在不同的点调用任意代码——早期是在对命令行参数进行分析之后，但在对它们进行任何操作之前；后期几乎是编译开始之前的最后一件事，即在完成所有命令行参数处理之后，等等。目前，您可以选择编译是在每一点停止还是继续，但不必更改驱动程序所做的任何操作。您可以为以后记录一些信息，或者执行自己的其他操作。

`some_input`和`no_input`给您一个修改编译器主输入的机会（通常输入是一个包含箱子顶部模块的文件，但也可以是一个字符串）。您可以记录输入或执行自己的其他操作。

忽略`parse_pretty`很不幸，希望能有所改善。有一个默认的实现，所以您可以假装它不存在。

`build_controller`返回A`CompileController`对象对于更细粒度的编译控制，下面将对其进行描述。

我们将来可能会增加更多的选择。

### `CompilerController`

`CompilerController`是由以下内容组成的结构`PhaseController`S和旗帜。目前，只有标志，`make_glob_map`它指示是否生成全局导入的映射（由save analysis和可能的其他工具使用）。会话中可能存在应移动到此处的标志。

有一个`PhaseController`对于上述编译摘要中描述的每个阶段（并且我们可以在将来为更细粒度的控制添加更多）。他们都是`after_`一个阶段，因为它们在一个阶段结束时被检查（同样，这可能会改变），例如，`CompilerController::after_parse`控制分析后（以及宏扩展前）立即发生的操作。

各`PhaseController`包含一个名为`stop`它指示编译应该停止还是继续，以及在阶段指示的点执行回调。无论编译是否继续，都将调用回调。

有关编译状态的信息在`CompileState`对象。这包含编译器拥有的所有信息。请注意，此状态信息是不可变的-回调只能使用编译器状态执行代码，它不能修改状态。（如果有需求，我们可以改变）。回调的可用状态取决于编译期间调用回调的位置。例如，解析之后有一个ast，但没有语义分析（因为ast尚未分析）。翻译后，有翻译信息，但没有AST或分析信息（因为这些信息已被消耗/遗忘）。

## 一个例子-愚蠢的统计数据

我们的示例工具非常简单，它只是收集一些关于程序的简单而不是非常有用的统计信息；它被称为愚蠢的统计信息。您可以在下面的示例中找到完整的源代码（注释较多）[Github](https://github.com/nick29581/stupid-stats/blob/master/src). 要建造，就做吧`cargo build`. 在文件上运行`foo.rs`做`cargo run
foo.rs`（假设你有一个叫做`foo.rs`. 您还可以传递通常传递给rustc的任何命令行参数）。当您运行它时，您将看到类似于

```text
In crate: foo,

Found 12 uses of `println!`;
The most common number of arguments is 1 (67% of all functions);
25% of functions have four or more arguments.
```

为了使事情更简单，当我们谈论函数时，我们排除了方法和闭包。

您还可以使用可执行文件作为rustc的替代品，因为毕竟，这是本练习的重点。因此，无论在makefile设置中如何使用rustc，都可以使用`target/stupid`（或者你最终得到的任何可执行文件）相反。这可能意味着设置一个环境变量，也可能意味着将可执行文件重命名为`rustc`并设定你的道路。同样，如果使用cargo，则需要将可执行文件重命名为rustc并设置路径。或者，您应该能够使用[多锈](https://github.com/brson/multirust)绕过所有的路径（尽管我还没有尝试过）。

（请注意，此示例打印到stdout。我不完全确定在不同的情况下，来自Rustc的stdout能做什么。如果看不到任何输出，请尝试插入`panic!`后`println!`如果出错，那么货物应该将愚蠢的统计标准转储到货物标准输出）。

让我们从`main`我们的工具的功能非常简单：

```rust,ignore
fn main() {
    let args: Vec<_> = std::env::args().collect();
    rustc_driver::run_compiler(&args, &mut StupidCalls::new());
    std::env::set_exit_status(0);
}
```

第一行获取任何命令行参数。第二行使用这些参数调用编译器驱动程序。最后一行设置程序的退出代码。

唯一有趣的是`StupidCalls`对象我们传递给驱动程序。这是我们对`CompilerCalls`特点和是什么将使这个工具不同于鲁斯特克。

`StupidCalls`是一个主要为空的结构：

```rust,ignore
struct StupidCalls {
    default_calls: RustcDefaultCalls,
}
```

这个工具非常简单，不需要在这里存储任何数据，但通常您会这样做。我们嵌入了`RustcDefaultCalls`当我们需要与rust编译器完全相同的行为时，在IMPL中委托给的对象。大多数情况下，你不想在工具中这样做（或者至少不需要这样做）。但是，货物呼叫Rustc`--print file-names`，所以我们委托`late_callback`和`no_input`为了让货物快乐。

剩下的大部分`CompilerCalls`琐碎：

```rust,ignore
impl<'a> CompilerCalls<'a> for StupidCalls {
    fn early_callback(&mut self,
                        _: &getopts::Matches,
                        _: &config::Options,
                        _: &diagnostics::registry::Registry,
                        _: ErrorOutputType)
                      -> Compilation {
        Compilation::Continue
    }

    fn late_callback(&mut self,
                     t: &TransCrate,
                     m: &getopts::Matches,
                     s: &Session,
                     c: &CrateStore,
                     i: &Input,
                     odir: &Option<PathBuf>,
                     ofile: &Option<PathBuf>)
                     -> Compilation {
        self.default_calls.late_callback(t, m, s, c, i, odir, ofile);
        Compilation::Continue
    }

    fn some_input(&mut self,
                  input: Input,
                  input_path: Option<Path>)
                  -> (Input, Option<Path>) {
        (input, input_path)
    }

    fn no_input(&mut self,
                m: &getopts::Matches,
                o: &config::Options,
                odir: &Option<Path>,
                ofile: &Option<Path>,
                r: &diagnostics::registry::Registry)
                -> Option<(Input, Option<Path>)> {
        self.default_calls.no_input(m, o, odir, ofile, r);

        // This is not optimal error handling.
        panic!("No input supplied to stupid-stats");
    }

    fn build_controller(&mut self, _: &Session) -> driver::CompileController<'a> {
        ...
    }
}
```

我们不为任何回调做任何事情，也不更改用户提供的输入。如果他们没有，我们只是`panic!`这是处理错误的最简单方法，但不是非常用户友好，真正的工具会给出建设性消息或执行默认操作。

在`build_controller`我们建造我们的`CompileController`. 我们只想分析，我们想在扩展之前检查宏，所以我们在第一个阶段（解析）之后停止编译。在这个阶段之后的回调是工具通过遍历AST来完成实际工作的地方。我们通过创建一个AST访问者并让它从顶部（箱子根）遍历AST来实现这一点。当我们走过箱子后，我们会打印收集到的数据：

```rust,ignore
fn build_controller(&mut self, _: &Session) -> driver::CompileController<'a> {
    // We mostly want to do what rustc does, which is what basic() will return.
    let mut control = driver::CompileController::basic();
    // But we only need the AST, so we can stop compilation after parsing.
    control.after_parse.stop = Compilation::Stop;

    // And when we stop after parsing we'll call this closure.
    // Note that this will give us an AST before macro expansions, which is
    // not usually what you want.
    control.after_parse.callback = box |state| {
        // Which extracts information about the compiled crate...
        let krate = state.krate.unwrap();

        // ...and walks the AST, collecting stats.
        let mut visitor = StupidVisitor::new();
        visit::walk_crate(&mut visitor, krate);

        // And finally prints out the stupid stats that we collected.
        let cratename = match attr::find_crate_name(&krate.attrs[]) {
            Some(name) => name.to_string(),
            None => String::from_str("unknown_crate"),
        };
        println!("In crate: {},\n", cratename);
        println!("Found {} uses of `println!`;", visitor.println_count);

        let (common, common_percent, four_percent) = visitor.compute_arg_stats();
        println!("The most common number of arguments is {} ({:.0}% of all functions);",
                 common, common_percent);
        println!("{:.0}% of functions have four or more arguments.", four_percent);
    };

    control
}
```

这就是创建自己的嵌入式编译器替换或自定义编译器所需的全部内容！为了完整起见，我将介绍其他愚蠢的统计工具。

```rust
struct StupidVisitor {
    println_count: usize,
    arg_counts: Vec<usize>,
}
```

这个`StupidVisitor`结构只跟踪`println!`它所看到的和每个参数数目的计数。它实现`syntax::visit::Visitor`走过去。大多数情况下，我们只使用默认方法，这些方法在AST中不执行任何操作。我们超越`visit_item`和`visit_mac`要在进入项（项包括函数、模块、特性、结构等）和宏时实现自定义行为，我们只对函数感兴趣：

```rust,ignore
impl<'v> visit::Visitor<'v> for StupidVisitor {
    fn visit_item(&mut self, i: &'v ast::Item) {
        match i.node {
            ast::Item_::ItemFn(ref decl, _, _, _, _) => {
                // Record the number of args.
                self.increment_args(decl.inputs.len());
            }
            _ => {}
        }

        // Keep walking.
        visit::walk_item(self, i)
    }

    fn visit_mac(&mut self, mac: &'v ast::Mac) {
        // Find its name and check if it is "println".
        let ast::Mac_::MacInvocTT(ref path, _, _) = mac.node;
        if path_to_string(path) == "println" {
            self.println_count += 1;
        }

        // Keep walking.
        visit::walk_mac(self, mac)
    }
}
```

这个`increment_args`方法增加正确的计数`StupidVisitor::arg_counts`. 我们走完了之后，`compute_arg_stats`做一些基本的数学计算出我们想要的关于参数的统计数据。

## 接下来呢？

这些API是相当新的，而且在它们真正变好之前还有很长的路要走。如果你想看到一些改进或者你想做的事情，请在评论中告诉我或者[吉特布问题](https://github.com/rust-lang/rust/issues). 尤其是，我不清楚到底需要什么样的额外灵活性。如果您有适合此设置的现有工具，请试用它，如果有问题请通知我。

如果可能的话，可以看到RustDoc转换为使用这些API（虽然从长远来看，我更希望看到RustDoc在保存分析的输出上运行，而不是自己进行分析）。编译器的其他部分（例如，漂亮的打印、测试）可以重构为在内部使用这些API（我已经更改了要使用的保存分析）`CompilerController`）我一直在试验一个原型rustfmt，它也使用这些API。

[stupid-stats]: https://github.com/nrc/stupid-stats
