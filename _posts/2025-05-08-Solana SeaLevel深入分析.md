---
title: 深入分析 Solana SeaLevel
categories: [dev]
---



#### 交易结构

​	Solana 的交易在执行前必须声明它将访问哪些状态。交易会列出它在执行过程中访问的账户，并明确它是对账户进行读取还是写入。基本的 Solana 交易结构如下：

```json
"transaction": {
    "message": {
      "header": {
        "numReadonlySignedAccounts": ,
        "numReadonlyUnsignedAccounts": ,
        "numRequiredSignatures":
      },
      "accountKeys": [], //a list of accounts that the transaction will use
      "recentBlockhash": , //a recent blockhash
      "instructions": [//list of the instructions within the transaction
        {
          "accounts": [], //expanded below
          "data": "3Bxs4NN8M2Yn4TLb", //eBPF bytecode that contains the //instruction logic
          "programIdIndex": 2, //index of program being called in the //"accountKeys" array
          "stackHeight": null
        }
      ],
      "indexToProgramIds": {}
    },
    "signatures": [] //an array of signatures
  }
```

​	位于指令中的`accounts`列出了这个交易相关的所有账户，这个列表的元素是`AccountMeta`类型，该类型有3个字段，`pubkey` 是账户ID，`isSigner`表明该账户是否进行签名， `isWritable`标记在交易执行中是否可写，例如：

```json
"accounts": [
    {
      "pubkey": "3z9vL1zjN6qyAFHhHQdWYRTFAcy69pJydkZmSFBKHg1R",
      "isSigner": true,
      "isWritable": true
    },
    {
      "pubkey": "BpvxsLYKQZTH42jjtWHZpsVSa7s6JVwLKwBptPSHXuZc",
      "isSigner": false,
      "isWritable": true
    }
  ],
```

​	根据`AccountMeta`，TPU可以识别哪些账户在交易中会被写入，从而并行安排不冲突的交易执行。

#### 交易生命周期

Solana 交易的生命周期与大多数区块链相同，主要的差异是 Solana 没有内存池。简要概述一下交易从创建到提交的过程。

- 创建后，交易被序列化、签名并转发至 RPC 节点。
- 当 RPC 节点接收数据包时，它们会被 `SigVerify`，这一步检查签名是否与消息匹配。
- 在 `SigVerify` 后，数据包进入 `BankingStage` 这个阶段要么处理、要么转发。
- 在 Solana 上，节点知道 *leader schedule* 预生成的计划表，指明哪些验证者在哪些插槽成为领导者。一个领导者分配 4 个连续的插槽。基于此信息，非领导者节点会将交易直接转发给当前领导者和未来$X$（agave 中$X$是2）个领导者。注意，这不是共识破坏的改变，只是安排节点把交易转发给更多或更少的领导者。

<img src="https://img.learnblockchain.cn/2025/03/08/20_wwtupm.png" alt="20_wwtupm" style="zoom: 33%;" />

- 领导者从其他节点接收到数据包后，它会对数据包运行 `SigVerify`，并将数据包发送至 `BankingStage`。
- 在 `BankingStage` 中，交易被反序列化、调度和执行。
- `BankingStage` 还与 POH 组件进行通信，把交易批次Hash加入POH 时间戳，并且不断构建shreds并在进行转发。
- 其他节点接收到 shreds 后，它们会重放这些数据，当组成一个区块的所有 shreds 被接收时，验证者将比较执行结果与领导者提议的结果。
- 然后，验证者签署包含块哈希的消息并发送其投票。投票可以作为交易传递或gossip，但 Solana 选择将投票视为交易以加快共识。
- 当在一个分叉中达到验证者法定人数时，该分叉顶端的区块被确认。
- 当一个区块拥有 31 个或以上的确认区块时，该区块被 *rooted*，几乎不可能被重组。

#### PoH

​	Solana 的历史证明（Proof of History, PoH）是一种“去中心化的时钟”。PoH 的工作原理基于递归的 SHA-256 哈希，它用这种循环次数来表征时间的流逝。

其思想是基于SHA-256 哈希函数的以下特性：

- 具有**抗原像性**，即想要得到某个消息 m 的哈希值 h 的唯一方法就是将哈希函数 H 应用于 m；
- 并且在任何高性能计算机上，计算该哈希值所需的时间**完全相同**。

由于这两个特性，节点通过递归地执行 SHA-256 哈希，可以根据已计算的哈希次数来一致地判断时间已经过去了多久。此外，验证过程高度可并行化，因为每个哈希值都可以独立地与下一个进行比对，而不依赖于之前哈希的结果。

​	正是由于这些特性，PoH 使得 Solana 节点在**无需同步或通信**的情况下就能对时间的流逝达成一致。PoH计算hash的代码片段如下：

```rust
self.hash = hashv(&[self.hash.as_ref(), mixin.as_ref()]);
```

`mix_in` 是一段任意信息（在此上下文中为交易哈希），它被附加到前一个哈希值之后，表明 `mix_in` 所代表的事件发生在该哈希计算之前，从而实现对该事件的时间戳标记。

​	与`Banking`阶段相关的 PoH 特定概念包括：

1. `ticks`：tick 是一个时间段度量，定义为计算 $X$ （目前为 12,500） 次哈希。tick 所在的哈希是链中的第 12500 个哈希。
2. `entry`：entry 是一个带时间戳的交易批次。Entries 被称为entries，因为它们是 Solana 提交交易到账本的方式。entry由三部分构成：有两种类型的 entry：tick entries 和 transaction entries。tick entry 不包含任何交易并在每个 tick 生成。transaction entries 包含一批不冲突的交易。
   1. `num_hashes`: 自上一个 entry 以来执行的哈希数量。
   2. `hash`: 是基于前一 entry的hash 完成 num_hashes 次哈希后的结果。

#### Entry 限制

​	有许多规则决定一个区块是否有效，即使它包含的所有交易都是有效的。可以在 [这里](https://apfitzge.github.io/posts/road-to-bankless/?ref=ghost-2077.arvensis.systems) 找到一些规则，但与本文相关的规则是 *entry* 限制。该规则规定，*entry* 中的所有交易必须是非冲突的。如果一个 entry 包含冲突的交易，则整个区块无效并被验证者拒绝。通过强制所有数据包中的交易为非冲突的，验证者可以并行重放 entry 中的所有交易，而无需调度的开销。

然而，SIMD-0083 提出了解除这一限制，因为它限制了区块生成，并阻碍了 Solana 的异步执行。Andrew Fitzgerald 在他的这篇文章中讨论了这一限制及其认为 Solana 在朝间接执行的道路上的下一步。

​	需要明确的是，这个限制并不能完全决定交易如何调度，因为已执行的交易不必包括在下一个 entry 中，但它对于当前调度程序设计是一个重要的考虑因素。

#### Baking Stage

​	`BankingStage` 位于 `SigVerify` 模块与 `BroadcastStage` 之间，PoH 模块则与它并行运行。如前所述，`SigVerify` 是验证交易签名的模块。`BroadcastStage` 则将处理后的交易数据通过 `Turbine` 广播到网络中其他节点，而 `TPU_Forwarding` 模块负责将已净化的交易数据包传播给领导者节点。

<img src="https://img.learnblockchain.cn/2025/03/08/2_cfxjqv.png" alt="2_cfxjqv" style="zoom:33%;" />

​	在 `BankingStage` 中，来自 `SigVerify` 的交易数据包被缓冲并通过channel接收。投票交易用 `Tpu_vote_receiver` 和 `gossip_vote_receiver` 两个管道处理，而非投票交易则由 `non_vote_receiver` 接收。之后，数据包根据leader schedule进行转发或“消费”。如果节点不是领导者或即将成为leader，则净化后的数据包被转发到合适的节点。节点成为领导者时则“消费”这些交易数据包，这些数据包将被反序列化、调度并执行。

​	执行阶段中，一批交易被调度后，TPU 将：

- 运行检查：工作线程将检查交易：
  - 是否过期；通过检查引用的 *blockhash* 是否过于陈旧。
  - 是否已经包含在之前的区块中。
- 获取锁：工作线程尝试为批次中的所有交易获取读写锁。如果锁获取失败，稍后再尝试执行交易。
- 加载账户并验证签名者是否可以支付费用：线程检查加载的程序是否有效，加载执行所需的账户并检查签名者是否可以支付交易中指定的费用。
- 执行：工作线程创建 VM 实例并执行交易。
- 进行PoH：执行结果和交易IDs发送到 PoH 线程，PoH 线程将生成一个 *entry*；同时，*entries* 也会发送到 BroadcastStage。
- 提交：如果PoH记录成功，则更新状态并提交结果。
- 解锁：解除账户的锁。

#### 中央调度器

- 交易来自 SigVerify ，并根据是否为投票事务进行拆分。
- 非投票交易转移到一个缓冲区，按优先级进行排序。通常按如下规则进行优先级排序：
  - 是否包含小费（tip account）或 MEV payment。
  - 交易类型（如高频 DEX 操作可能优先）。
  - 时间戳或账户热点评分。
- 中央调度器扫描最高优先级的交易，对其进行过滤，交易会被**插入到一个 Account Access Graph（账户访问图）中**，该图的拓扑结构反映了哪些交易可以并发执行、哪些必须串行。。
- 当达到目标大小或全局队列为空时，中央调度线程开始以避免线程间冲突的方式将交易调度到工作线程。
- 调度的交易批次将像以前一样执行。

中央调度器架构

<img src="https://img.learnblockchain.cn/2025/03/08/1f_cdcwyt.png" alt="1f_cdcwyt" style="zoom:33%;" />

##### 调度算法 - Prio-Graph

​	在调度过程中，最高优先级的交易会从队列中弹出，并被插入到一个**懒惰填充的依赖图（prio-graph）**中。在这个图中，边的存在表示：交易与下一个最高优先级交易之间存在**账户读写冲突**。开始调度时，会从优先级队列中取出**“前瞻窗口”（look-ahead window）**大小的交易批次，并插入图中。调度器进行交易调度时，它会**继续从优先队列中取出新的交易并插入图中**。图结构允许我们可以**预判目前看似不冲突的交易，是否会与后续交易发生冲突**，即存在“图连接（graph join）”。如果检测到图连接，调度器可以**将这些交易放在同一个线程中执行**，以避免创建无法调度的交易（unschedulable transaction）。目前需要**调优“前瞻窗口”的大小**，如果窗口太小，会导致：

- 太多交易之间的依赖关系在图中看不到，进而使得某些交易变成**不可调度状态**；
- 最终导致延迟增加。

如果窗口太大，会使得图中看到太多“潜在冲突”，导致：

- 太多交易预防性地被绑定到同一个线程；
- 最终**失去并行执行的优势**。
