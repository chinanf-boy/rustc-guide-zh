# 关于编译团队

rustc由维护[Rust编译团队][team]。属于该团队的人员共同致力于跟踪回归并实施新功能。Rust编译团队的成员是对rustc及其设计做出重大贡献的人。

[team]: https://www.rust-lang.org/governance/teams/language-and-compiler

## 讨论

目前编译器团队在很多地方聊天。有一个持续的[线]在内部委员会关于试图找到一个永久的家。无论如何，您现在可以在三个地方之一找到人：

-   该`#rustc`mozilla的IRC频道（`irc.mozilla.org`）
-   该`t-compiler`流[Zulip实例](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler)
-   该`compiler`通道上[生锈的不和谐](https://discord.gg/rust-lang)

## Rust编译器会议

编译器团队每周召开一次会议，我们会对其进行分类，并尝试保持对新错误，回归和其他内容的掌握。这个会议的总体计划可以在[rust-compiler-meeting etherpad][etherpad]。它的工作原理大致如下：

-   **回顾P-high错误：**P-high bug对于我们积极跟踪进度非常重要。理想情况下，P-high bug应始终有受让人。
-   **查看新的回归：**然后，我们寻找新的情况，编译器破坏了以前工作的代码。回归几乎总是标记为P-高;主要的例外是错误修复（尽管我们经常这样做[旨在首先发出警告][procedure]）。
-   **检查I提名的问题：**这些是需要团队反馈的问题。
-   **检查测试版提名：**这些是向后推向be​​ta的事物的提名。

会议目前在波士顿时间周四上午10点举行（典型的是UTC-4，但夏令时有时会让事情变得复杂）。

会议通过“聊天媒体”举行 - 过去曾是IRC，但我们目前正在评估其他替代方案。检查[EtherPad的]找到当前的家（并看到[这个内部线程][thread]对于一些持续的讨论）。

[etherpad]: https://public.etherpad-mozilla.org/p/rust-compiler-meeting

[thread]: https://internals.rust-lang.org/t/where-should-the-compiler-team-and-perhaps-working-groups-chat/7894

[procedure]: https://forge.rust-lang.org/rustc-bug-fix-procedure.html

## 团队成员资格

当有人在一段时间内对编译器做出重大贡献时，通常会提供Rust团队的成员资格。成员资格既是承认也是义务：编译团队成员通常希望帮助维护，以及进行评论和其他工作。

如果您有兴趣成为编译器团队成员，首先要做的是开始修复一些错误，或参与工作组。找到错误的一个好方法是寻找[用E-easy标记的未解决问题](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AE-easy)要么[E-导师](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AE-mentor)。

### r +权利

一旦你制作了一些个人PR，我们通常会提供r +特权。这意味着您有权指示“bors”（管理哪些PR落入rustc的机器人）合并PR（[这里有一些关于如何与bors交谈的说明][homu-guide]）。

[homu-guide]: https://buildbot2.rust-lang.org/homu/

审稿人的指导原则如下：

-   无论分配给谁，欢迎您随时查看任何PR。但是，除非：
    -   您对该部分代码充满信心。
    -   您确信没有其他人愿意先审查它。
        -   例如，有时人们会表达在PR降落之前审查PR的愿望，可能是因为它触及了代码中特别敏感的部分。
-   在审核时始终保持礼貌：您是Rust项目的代表，所以当涉及到这个项目时，您应该会超越它。[行为守则]。

[code of conduct]: https://www.rust-lang.org/policies/code-of-conduct

### 举手击掌

拥有r +权限后，您还可以添加到高五轮换。high-five是将传入的PR分配给审阅者的机器人。如果添加，您将被随机选中以查看PR。如果您发现您被分配了一个您觉得不舒服的PR，您也可以发表评论`r? @so-and-so`分配给其他人 - 如果你不知道要求谁，只需写`r?
@nikomatsakis for reassignment`和@nikomatsakis会为你挑选一个人。

[hi5]: https://github.com/rust-highfive

获得高五者名单非常受欢迎，因为它降低了我们所有人的审核负担！但是，如果您没有时间及时向人们提供有关其PR的反馈，那么您最好不要列入清单。

### 完整的团队成员

一旦有人对Rust编译器做出了很多贡献，通常会扩展完整的团队成员资格，理想情况（但不一定）是多个领域。有时这可能是实现一个新功能，但它也很重要 - 也许更重要！- 有时间和意愿帮助进行一般维护，例如错误修正，跟踪回归和其他不那么有魅力的工作。
