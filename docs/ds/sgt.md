author: Ir1d, 0xis-cn

## 引入

**替罪羊树** 是一种依靠重构操作维持平衡的重量平衡树。替罪羊树会在插入、删除操作后，检测树是否发生失衡；如果失衡，将有针对性地进行重构以恢复平衡。

一般地，替罪羊树不支持区间操作，且无法完全持久化；但它具有实现简单、常数较小的优点。

## 基本结构和操作

替罪羊树的核心操作是重构、插入和删除操作。

### 节点信息

替罪羊树需要存储以下信息，用于树的自平衡操作：

-   树的结构信息：
    -   `id`：已使用节点数目；
    -   `rt`：根节点；
    -   `lc[x]`，`rc[x]`：左、右子节点；
    -   `tot[x]`：以 $x$ 为根的子树大小（每个节点计数为 $1$）[^tot-cnt]；
    -   `tot_active`：整个树中未删除（即 `cnt[x] != 0`）的节点的数目。

当使用替罪羊树实现平衡树时，还需要存储如下信息：

-   平衡树的节点信息：
    -   `val[x]`：节点存储的值；
    -   `cnt[x]`：节点存储的值的计数（可能为 $0$）；
    -   `sz[x]`：以 $x$ 为根的子树存储的值的计数。

为了维护节点信息，可以实现 `push_up` 操作：

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:10:14"
    ```

应注意 `tot[x]` 和 `sz[x]` 的更新方式的不同。

### 重构操作

当树发生失衡时，需要对某个子树进行重构，使之尽可能平衡。重构分为两步：

-   对要重构的子树做中序遍历，将所有未删除节点存到序列中；
-   二分建树，即取中点为根，左右两侧递归地建子树，并更新节点信息。

参考实现如下：

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:16:40"
    ```

建树时注意维护节点信息，包括叶子节点的信息。

单次重构的复杂度是 $\Theta(|T_x|)$ 的，因此如果每次插入、删除时都进行重构，复杂度将难以接受。替罪羊树的核心思想就在于对重构时机的选择，进而实现了 $O(\log n)$ 的均摊复杂度。

### 插入操作

插入操作时，可能会引起树的失衡。为了判断树的失衡，需要引入参数 $\alpha\in(0.5,1)$，通常的选择在 $0.7\sim 0.8$ 之间。

如果新插入的节点的深度超过了 $\lfloor\log_{1/\alpha}|T|\rfloor$，其中，$|T|$ 为更新后的树的大小，就需要在回溯时寻找失衡发生的节点并进行重构。此时，需要根据如下条件判断以 $x$ 为根的子树失衡：

$$
\max\{|T_{\mathrm{left}(x)}|,|T_{\mathrm{right}(x)}|\} > \alpha\cdot |T_x|,
$$

其中，$\mathrm{left}(x)$ 和 $\mathrm{right}(x)$ 分别为 $x$ 的左、右子节点，$|T_x|$ 为以 $x$ 为根的子树大小。

插入操作的具体步骤如下：

-   首先利用二分查找树的性质向下找到插入值的位置，下探时记录深度；
-   如果已经有节点，直接修改节点信息，否则新建节点；
-   如果新建节点过深，就需要自下而上回溯到根，更新节点信息，并记录第一个（或任意一个）子树失衡的节点；
-   如果存在失衡节点，重构失衡节点的子树。

参考实现如下：

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:42:67"
    ```

注意，单次插入至多引起一次重构。如果没有新增节点或是新增节点并没有过深，又或是本次回溯过程中已经执行过重构，就不需要继续判断失衡了。多余的重构可能会导致效率损失[^insert-complexity]。回溯过程中的第一个失衡节点，就是所谓的「替罪羊」。

### 删除操作

删除操作的处理则非常简单。替罪羊树的删除策略是「懒删除」，即节点为空时，不移除节点，而是留待后续处理。

当然，如果树中空节点过多，树的访问效率会大大下降。因此，替罪羊树维护两个计数，整个树中未删除节点的数目和整个树实际使用的节点数目。对于选定的阈值[^threshold] $\alpha\in(0,1)$，当前者与后者的比值下降到 $\alpha$ 以下时，就对整个树做一次重构，重构时删除所有空节点。

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:69:96"
    ```

### 时间复杂度

大小为 $n$ 的替罪羊树的访问节点的时间复杂度为单次 $O(\log n)$ 的，$\Theta(n)$ 次插入和删除的均摊时间复杂度也是单次 $O(\log n)$ 的。

本节对替罪羊树的时间复杂度仅做简要论证，详细证明请参考原论文。

??? note "替罪羊树的时间复杂度的论证"
    由于采用懒删除的策略，未删除节点数目为 $n$ 的替罪羊树可能占用了 $\alpha^{-1}n$ 个节点。由于仅仅相差一个常数因子，本文在表述中并不区分替罪羊树的未删除节点数目和占用节点数目，而统一称为「树的大小」。
    
    1.  **访问操作**：访问操作的复杂度得以保证，是因为大小为 $n$ 的替罪羊树的树高总是 $O(\log n)$ 的。
    
        首先，区分两个概念：
    
        -   $\alpha$‑重量平衡：所有节点处，左、右子节点的子树的大小均不超过该节点处子树的大小的 $\alpha$ 倍；
        -   $\alpha$‑高度平衡：树的高度不超过 $\lfloor\log_{1/\alpha}|T|\rfloor$，其中，$T$ 是树的大小。
    
        $\alpha$‑重量平衡可以推出 $\alpha$‑高度平衡，因为子节点深度每增加一，大小就减少到原来的 $\alpha$ 倍；反过来则不一定成立。更严格地说，每次操作结束后，替罪羊树都总是 $\alpha$‑高度平衡的[^hei-bal]，这就保证了访问操作的复杂度。
    
        只有插入操作会改变树的结构，所以只需要说明每次插入操作后，替罪羊树都仍是 $\alpha$‑高度平衡的。如果新插入的节点过深，造成了整个树不再 $\alpha$‑高度平衡，那么自该节点回溯至根时，至少会碰上一个节点，即「替罪羊」，它的子树不再 $\alpha$‑重量平衡。将其重构后，子树的高度将降低至少一，故而新插入的节点将不再过深。
    2.  **插入操作**：插入操作的复杂度是均摊 $O(\log n)$ 的。
    
        设在某次插入操作后，节点 $x$ 处发生一次子树的重构，时间成本为 $\Theta(|T_x|)$。因为节点 $x$ 刚插入时，或者它刚刚经历了（自身或祖先节点的）上一次重构之后，它的左右子树至多只相差一个节点。而在这次重构之前，节点 $x$ 处必然成立
    
        $$
        \max\{|T_{\mathrm{left}(x)}|,|T_{\mathrm{right}(x)}|\} > \alpha\cdot |T_x|.
        $$
    
        这一条件保证左右子树的大小的差值至少为 $(2\alpha-1)|T_x|$ 的。因而，这两次重构之间，子树 $T_x$ 中插入了 $\Omega(|T_x|)$ 个节点。
    
        利用摊还分析可知[^alternative-analysis]，如果每次插入节点时，都在（可能的重构前）自根到该节点的路径上的每个节点都增加 $\Theta(1)$ 的势能，那么到节点 $x$ 处子树重构前，必然已经在节点 $x$ 处累积了 $\Omega(|T_x|)$ 的势能，足以用于偿还 $x$ 处子树重构的成本 $\Theta(|T_x|)$。因为树的深度都是 $O(\log n)$ 的，所以单次插入增加的势能是 $O(\log n)$ 的；这说明，$\Theta(n)$ 次插入操作中势能增加的总和是 $O(n\log n)$ 的。由此，子树重构的总成本也是 $O(n\log n)$ 的，单次插入操作（含重构）的均摊时间复杂度就是 $O(\log n)$ 的。
    
        注意，分析中没有假定在节点 $x$ 处的两次重构之间，子树 $T_x$ 内部没有发生其它的重构。因此，只要只重构满足失衡条件的节点处的子树，就能保证复杂度正确。
    3.  **删除操作**：删除操作的复杂度也是均摊 $O(\log n)$ 的。
    
        删除引起的重构会导致整个树不含空节点。而某次删除引起重构之前，整个树中已经有 $\Theta(n)$ 个空节点，这意味着至少进行了 $\Theta(n)$ 次删除操作。因为每次删除操作的寻址的复杂度是 $O(\log n)$ 的，且单次重构的复杂度是 $\Theta(n)$，所以这 $\Theta(n)$ 次删除操作的实际时间成本为
    
        $$
        \Theta(n)O(\log n)+\Theta(n)
        $$
    
        的。故而，单次删除的均摊复杂度为 $O(\log n)$ 的。

## 平衡树操作

本节介绍用替罪羊树维护可重集的方法。

除上节介绍的操作外，其余操作均为平衡树的常见操作。但是，因为替罪羊树中可能存在空节点，这些操作也需要相应调整。

### 查询排名

利用二分查找树的性质向下查找节点位置，过程中记录路径左侧存储的值的数目即可。

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:98:111"
    ```

### 根据排名查询值

利用节点记录的子树存储值的数目信息向下查找即可。注意可能存在计数为零的节点。

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:113:128"
    ```

### 查询前驱、后继

以上两种功能结合即可。

???+ example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp:130:134"
    ```

如果想直接实现，应注意处理计数为零的节点。

### 参考实现

本节的最后，给出模板题 [普通平衡树](https://loj.ac/p/104) 的参考实现。

??? example "参考实现"
    ```cpp
    --8<-- "docs/ds/code/sgt/sgt.cpp"
    ```

## 参考资料

-   Galperin, Igal, and Ronald L. Rivest. "Scapegoat trees." Proceedings of the fourth annual ACM-SIAM Symposium on Discrete algorithms. 1993.
-   [Scapegoat Tree - Wikipedia](https://en.wikipedia.org/wiki/Scapegoat_tree)
-   [替罪羊树 - riteme 的博客](https://riteme.site/blog/2016-4-6/scapegoat.html)

[^tot-cnt]: 也可以只统计未删除节点数目，此时不再需要统计 `tot_active`，而需要统计所有占用节点数目 `tot_max`，代码相应调整即可。

[^insert-complexity]: 根据后文的复杂度分析可知，这些效率损失仅意味着更大的常数因子，而复杂度依然是正确的。因为判断树深可能会涉及较多的浮点数对数运算，不判断树深只判断失衡的代码在某些数据中可能更快。

[^threshold]: 不必与上文插入操作时选取的参数相同。尽管原论文做了这样的假定，但是选取不同的参数只会导致单次操作的复杂度中常数项的变化，整体复杂度依然是正确的。

[^hei-bal]: 按原文定义，$n$ 指未删除节点的数目，故而只能保证树高不超过 $\lfloor\log_{1/\alpha}n\rfloor+1$，这称为弱 $\alpha$‑高度平衡。此处没有细究该常数项的差异。

[^alternative-analysis]: 有些文章会简单分析成 $\Omega(|T_x|)$ 次插入对应一次重构，故而均摊复杂度为 $\dfrac{\Omega(|T_x|)O(\log n)+\Theta(|T_x|)}{\Omega(|T_x|)} = O(\log n)$。这样的思路可以辅助理解均摊复杂度为什么正确，但并不严谨。这是因为，一次插入可能对应着多个祖先节点的重构，故而当节点 $x$ 发生重构时，子树内未引起重构的节点数目并不显然是 $\Omega(|T_x|)$ 的。
