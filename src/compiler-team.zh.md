# 关于编译团队

rustc 由[Rust 编译团队][team]维护。属于该团队的人员共同致力于,跟踪性能退步，并实现新功能。Rust 编译团队的成员是对 rustc 及其设计做出重大贡献的人。

[team]: https://www.rust-lang.org/governance/teams/language-and-compiler

## 讨论

目前编译器团队在很多地方聊天。在内部版块有一个长期[thread]，关于试图找到一个永久的家。无论如何，您现在可以在三个地方之一，找到人：

- mozilla 的 IRC 的`#rustc`频道（`irc.mozilla.org`）
- [Zulip 实例](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler)的`t-compiler`平台
- [rust-lang discord](https://discord.gg/rust-lang)的`compiler`频道

## Rust 编译器会议

编译器团队每周召开一次会议，我们会对其进行分类，并尝试保持对新错误，性能退步和其他内容的掌握。这个会议的总体计划可以在[rust-compiler-meeting etherpad][etherpad]。它的工作原理大致如下：

- **回顾 P-high 错误：**P-high bug 对于我们积极跟踪进度非常重要。理想情况下，P-high bug 应始终有人抗着。
- **查看新的性能退步：**然后，我们寻找新的情况，编译器破坏了以前工作的代码。性能退步几乎总是标记为 P-high; 主要的例外是错误修复（尽管我们经常，[在最开始发出警告][procedure]）。
- **检查 I-nominated 的问题：**这些是需要团队反馈的问题。
- **检查测试版本的提名：**这些是推进 be​​ta 的提名名单。

会议目前在波士顿时间周四上午 10 点举行（典型的是 UTC-4，但夏令时有时会让事情变得复杂）。

会议通过“聊天媒体”举行 - 过去曾是 IRC，但我们目前正在评估其他替代方案。查看[EtherPad]找到当前的家，（并看到[这个内部线程][thread]，有相应的讨论）。

[etherpad]: https://public.etherpad-mozilla.org/p/rust-compiler-meeting
[thread]: https://internals.rust-lang.org/t/where-should-the-compiler-team-and-perhaps-working-groups-chat/7894
[procedure]: https://forge.rust-lang.org/rustc-bug-fix-procedure.html

## 团队成员资格

当有人在一段时间内对编译器做出重大贡献时，通常会提供 Rust 团队的成员资格。成员资格既是承认也是义务：编译团队成员通常希望帮助维护，以及进行评论和其他工作。

如果您有兴趣成为编译器团队成员，首先要做的是开始修复一些错误，或参与工作组。找到错误的一个好方法是寻找[用 E-easy 标记](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AE-easy)要么[E-mentor](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AE-mentor)的未解决问题。

### r+ 权限

一旦你制作了一些个人 PR，我们通常会提供 r+ 权限。这意味着您有权指示“bors”（管理哪些 PR 落入 rustc 的机器人）合并 PR（[这里有一些关于如何与 bors 交谈的说明][homu-guide]）。

[homu-guide]: https://buildbot2.rust-lang.org/homu/

审稿人的指导原则如下：

- 无论分配给谁，欢迎您随时查看任何 PR。但是，除非：
  - 您对该部分代码充满信心。
  - 您确信没有其他人愿意先审查它。
    - 例如，有时人们会表示，想在 PR 降落之前，审查 它，可能是因为它触及了代码中特别敏感的部分。
- 在审核时始终保持礼貌：您是 Rust 项目的代表，所以当涉及到这个项目时，您应该会超越[行为守则]。

[code of conduct]: https://www.rust-lang.org/policies/code-of-conduct

### 举手击掌

一旦拥有 r+权限后，您还可以添加到 high-five 轮换。high-five 是将传入的 PR 分配给审阅者的机器人。如果添加，您将被随机选中以查看 PR。如果您发现您被分配了一个您觉得不舒服的 PR，您也可以发表`r? @so-and-so`评论，分配给其他人 - 如果你不知道要求谁，只需写`r? @nikomatsakis for reassignment`，拿 `@nikomatsakis` 会为你挑选一个人。

[hi5]: https://github.com/rust-highfive

high-five 获得者名单非常受欢迎，因为它降低了我们所有人的审核负担！但是，如果您没有时间及时向人们提供有关其 PR 的反馈，那么您最好不要参与近来。

### 完整的团队成员

一旦有人对 Rust 编译器做出了很多贡献，通常会给予完整的团队成员资格，理想情况（但不一定）是多个领域的努力。有时可能是实现一个新功能，但它也很重要 - 或是超级重要！- 有时间和意愿帮助进行一般维护，例如错误修正，跟踪性能退步和其他不那么有魅力的工作。
