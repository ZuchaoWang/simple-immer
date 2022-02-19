# simple-immer

[Immer in 100 lines](https://ahabhgk.github.io/blog/immer-in-100-lines) [[知乎](https://zhuanlan.zhihu.com/p/341250210)] [[掘金](https://juejin.cn/post/6912724366576844808/)]

## 代码解释

simple-immer 是 [ahabhgk](https://github.com/ahabhgk/simple-immer) 对 immer.js 的一个简单实现。它提供一个 produce 函数，能够将用户的 mutation 代码内部转化为 immutable 的方式来更新 state。

state 表示成一个 object，在概念上可以看做是一个树形结构。树的每个节点是一个 object，每个节点的子节点是这个 object 的 object 类型的 property。每个节点可以有 3 种状态：

- 原始状态：未被从父节点读出过，更未被改写过
- 已读状态：被父节点通过 get 操作读出，但未被改写过
- 已写状态：自己执行 set 操作改写过值，或者自己的后代节点执行过 set 操作

各种状态间的转化如下，后面会详细介绍

```text
        (createImmer/get):createCraft          set:markChanged
原始状态 -------------------------------> 已读状态 ----------------> 已写状态
   ^                                                                 |
   └-----------------------------------------------------------------┘
                           finish:finalize
```

原始状态的节点只需要记录原始数据，不需要任何其他信息。已写状态的节点需要通过 copy-on-write 的方式同时记录了原始值（base）和改写过的值（copy）。已读节点看似不需要记录任何状态，实际上需要记录一个父节点的指针（parent）；已写节点也需要这个指针。这是因为当我们 set 一个节点时，我们不仅需要将该节点变成已写状态，还需要确保所有祖先节点都变成已写状态。为了访问祖先节点，我们需要从当前节点到根节点都有 parent 指针。注意到这些节点都是已读或者已写状态的，因此我们干脆就要求这两种状态的节点都有 parent 指针。

已读和已写状态的节点统一用 draft 表示。draft 通过 createDraft 函数生成。draft 是通过 proxy 实现的，这样针对 draft 的所有 get/set 操作都会被截取，我们由此实现记录 parent 指针、copy-on-write 等功能。draft 有一个 state 变量，字段含义如下：

- base：原始值
- copy：改写过的值
- drafts：对于已读状态的节点，这里存储了所有已读状态的子节点；对于已写状态的节点，没有用处
- parent：父节点指针
- finalized：仅用于 finish 中的 finalize 阶段；记录该节点是否执行过 finalize 函数，以避免 finalize 函数因为自引用陷入无限循环

copy 字段可以区别已读和已写状态：已读状态的节点的 copy 值为 undefined，已写状态的节点的 copy 值存储了改写过的值。在 set 函数改写值之前，会从当前节点到根节点依此调用 markChanged 函数。该函数会将 base 和 drafts 浅拷贝到 copy 中，这样之后值的改写只会作用到 copy 上，不会改写 base。这就是 copy-on-write 的基本实现。

不同状态节点将最新的值存储在不同的地方。原始状态的节点未被改写，本身就是最新的值。已读状态的节点虽然未被改写，但是部分子节点已经被封装成了 draft 形式。因此我们首先在 drafts 中查找，然后再看 base。已写状态的的节点，最新值完全存储在 copy 中。

最后我们来看看 produce 函数。在执行之前，state 树形结构中所有节点都在原始状态。执行的过程分为 3 个步骤：

- createImmer，将 state 根节点转变为已读状态
- producer，对部分节点进行读写操作，部分节点会变成已读或者已写状态
- finish，汇总所有节点的最新值，返回新的 state；新 state 所有节点全部为原始状态

finish 函数有两个工作：

- 对 draft 进行 finalize，返回每个节点的最新值；这里针对已读状态的节点直接返回 base，完全不需要 drafts 了，这是因为后面不会再有 set 操作了，所以不需要 draft 中的 parent 指针了
- 通过对 proxy 进行 revoke，无效化所有的 draft

## 对代码的修改

- line 28：似乎不需要 `!isDraft(state)` 判断。该判断永远返回 true，因此我注释掉了。
