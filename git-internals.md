# Git Internals

本文会深入 Git 的实现，包括 Git objects（blob / tree / commit / tag / delta） 、Git references 和 Git packfiles。

## Glance at `.git` directory

当我们用 Git 来管理一个目录时，Git 会在其下创建一个 .git 目录来维护所有的元数据。一个典型的 .git 目录如下：

```bash
$ ls -F1
config
description
HEAD
hooks/
info/
objects/
refs/
```

其中，description 仅用于 GitWeb，config 保存了一些 project-specific 的配置信息，info/ 下的 exclude 文件保存了 .gitignore 中定义的 ignored patterns。hooks/ 保存了 client- 或者 service-side 的 hook scripts。以上这些概念不在下文讨论。

objects/ 用于存储所有 Git 对象的数据。refs/ 保存了 objects/ 中部分对象的 aliases。HEAD 为 refs/ 中某个 alias 的软链接（symbolic link）。

## Git Objects

Git 将 data 和 metadata 都抽象成对象来进行管理，每一对象会有一个哈希值（SHA1）来唯一标识。下面，我们基于需求一步一步引入不同的 Git 对象。

### Build a snapshot of source code tree with Tree and Blob object

给定一棵源码树 $T$，Git 很自然的定义了 tree 和 blob 对象来表达 $T$，如下图。其中，blob 用于保存文件的数据。tree 对象类似于 Unix FS 中的目录，由若干 entry 构成，每一个 entry 可指向 blob 或者 tree（通过保存对象的哈希值，除此之外还保存了对象的类型（blob / tree）、大小和文件类型（text / binary 等等））。

![](https://git-scm.com/book/en/v2/images/data-model-1.png)

如下图，Git 通过 tree 和 blob 对象给出了 $T$ 的一个 snapshot $V_x$，i.e. $T$ 的一个版本。如果，我们想访问 $V_x$ 中的某个文件，则只需要记住根节点的哈希值 3c4e9c…，再依次遍历取得相应的文件。

![](https://git-scm.com/book/en/v2/images/data-model-2.png)

### Link multiple versions with Commit object

下图中，$T$ 产生了三个版本 $V_0$ (tree d8329f)、$V_1$ (tree 0155eb)、$V_2$ (tree 3c4e9c)，且版本之间存在依赖关系，i.e. $V_0 \rightarrow V_1 \rightarrow V_2$。那么，我们该如何记录该关系？Git 通过引入 commit 对象解决这一问题。同时，commit 对象还保存了产生当前 commit 的 author 信息、版本生成理由等等。

另外，图中的版本演化很简单，是一个线性的 chain，即单 branch。事实上，commit 可以有多个 parent，从而可以表达复杂的版本衍生关系，即我们在大型项目中看到的错综复杂的 branches。

另外，源码树的不同版本之间的某一个文件没有变化，则引用同一个对象（Git 的对象化设计很容易实现这一点），如 $V_1$ 和 $V_2$ 都包含 text.txt（存储为 blob 对象，且哈希值为 1f717a...）文件且未经修改，故引用同一个 blob 对象。当然，若同一个文件存在变化（即使只有 1 字节），Git 也会为其生成一个新的 blob 对象，如下图中，相比 $V_0$ 中的 test.txt 文件，$V_1$ 仅仅将文件中的 1 改为 2，则 Git 在 $V_1$ 中为 test.txt 生成了一个新的 blob 对象 1f7a7a…，并指向它。

![](https://git-scm.com/book/en/v2/images/data-model-3.png)

### Human-friendly interaction with refs and tags

当我们需要访问源码树的某个版本时，需要知道其 commit 对象的哈希值，不方便。此时，我们可以给 commit 对象取一个别名（refs），然后通过该别名来查看（checkout）相应版本的源码树。这些别名都保存在 refs 目录下。逻辑上，refs 分为动态（dynamic）和静态（static）refs，分别对应于 branch refs 和 tags。

Branch 的 refs 总是指向某个 branch 最新版本的 commit，即随着 branch 新 commit 的添加而实时更新。其中，HEAD 指向当前工作区所用 branch 的 ref（一个工作目录只能 checkout 源码树的一个版本），如指向 refs/heads/master。

通过 tag，我们可以给 .git/objects 中任意一个 object 取一个别名。通常，我们会给特定的 commit 打上类似 v0.1、v0.2 的标签。当然，我们也可以给某个 blob 打上标签，如给 test.txt 对应的 blob: fa49b0... 打上标签 hello_test，则以后我们可以直接通过 hello_test checkout 位于 $V_0$ 版本中的 test.txt 文件。

另外，tags 分 lightweight 和 annonated。Annonated tag 也是一类 Git 对象。

![](https://git-scm.com/book/en/v2/images/data-model-4.png)

## Lower storage cost

Git 通过两类技术来减少存储开销：压缩和增量存储。首先 Git 存储的每一个对象都是先压缩再存储。另外，在我们将本地仓库 push 到 remote 仓库时，Git 会对仓库进行 GC 操作，即将所有文件打包成一个 packfile，并按照一定的算法将重复的文件进行增量存储。我们也可以通过显示命令 `git gc` 来执行这一过程。

### Pack heuristics

给定义一个仓库 $R_0$，逻辑上如上图，物理上，Git 会按对象进行存储。先考虑输入若干 Git 对象，如何紧凑的存储它们以减少存储开销？Git 采用增量存储方法，现给出算法。

简单的说，pack 算法会先对所有的 objects 排序，然后顺序扫描每一个 object $O$，同时将其之前的 W 个 objects 作为其 parent。记某一个 parent 为 $B$，通过分别计算和比较 $cost\_of\_delta(B, O)$ 以在 W 个 parent 中选取最优的那个 $B$ 作为 $O$ 的 base。举一个例子，考虑五个 Git 对象 $D_1,\ ...,D_5$，排序后得 $D_1, D_4, D_2, D_3, D_5$。$D_1$ 首先作为 root。对于 $D_4$，我们只能选择 $D_1$ 作为其 base。对于 $D_2$，我们需要在 $D_4$ 和 $D_1$ 之间选择。需要注意的是，若以 $D_4$ 作为 base，则会形成长 delta chain $D_1 \rightarrow D_4 \rightarrow D_2$。长的 delta chain 会导致 $D_2$ 的 retrieval 开销增大。故此时，我们有必要精心选择一个好的代价公式来帮助形成较优的增量存储方案。

See ref. 1 and 7 for more details.

### Content-aware diff. algorithm (adaptive xdiff)

See ref. 3 for more details.

### Pack format

Git 在 GC 后，会将涉及的所有对象打包成一个 packfile，同时生成一个方便二分搜索的 index 文件。Packfile 和 index 文件的格式如下图：

![](https://i.stack.imgur.com/TSOqy.png)

一个 packfile 由 header、body 和 trailer 构成。Packfile 的 body 中存储了所有的 Git 对象（注：增量数据也是一类 Git 对象）的数据，而对象大小可以是变长的，一般不超过 2GB。Index 文件中维护了 Git 对象的全局 ID （哈希值）和其在 packfile 中的 offset。如果我们想读取哈希值为 abcdef... 的 blob 对象，则需要先在 index 文件中以哈希值为 key 进行二分查找，取得 offset 后在 packfile 中读取相应的数据。若待读取对象被增量化，则通过对象中 base 字段记录的哈希值读取对应的对象数据并 merge，递归这一过程直至形成完整的对象。

See ref. 2 and 6 for more details.

### Why Git fail at managing large files?

目前 Git 的设计不能有效的处理以下三个问题：

1. 单个大文件（Git pack 中的 xdelta 算法需要将待处理数据完全加载到内存中，故单个文件的大小受到内存大小的制约）；
2. 文件数据很大；
3. 管理许多巨大的 packfiles。

当一个仓库中文件数量十分巨大时，Git 在 GC 或读取一个特定文件时效率会很低。同时，当仓库中存在多个 packfiles，Git 查询一个特定文件时的效率也会很低（需要在多个 index 文件中进行跳转，不利于二分搜索）。

See ref. 4 for more details.

## References

1. http://www.cs.umd.edu/~amol/DBGroup/2015/06/26/datahub.html
2. http://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#pack_file_format
3. https://stackoverflow.com/questions/9478023/is-the-git-binary-diff-algorithm-delta-storage-standardized
4. https://stackoverflow.com/questions/17888604/git-with-large-files/19494211#19494211
5. https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain
6. https://github.com/git/git/blob/master/Documentation/technical/pack-format.txt
7. https://github.com/git/git/blob/master/Documentation/technical/pack-heuristics.txt