架构起源
====================

.. note :: 本文档描述了 Hyperledger Fabric v1.0的最初架构提案。虽然从概念上讲，Hyperledger Fabric 实现是从架构方案开始的，但是在实现过程中还是修改了一些细节。最初的架构方案是按照最初的准备提出的。有关架构的更准确的技术描述，请参阅 `Hyperledger Fabric: A Distributed Operating System for Permissioned Blockchains <https://arxiv.org/abs/1801.10228v2>`__ 。

Hyperledger Fabric 架构有以下优势：

- **链码信任的灵活性**。该架构将链码(区块链应用程序)的*信任假设*与排序的信任假设分开。换句话说，排序服务可能由一组节点(排序节点)提供，并可以容忍其中一些节点出现故障或不当行为，而且每个链码的背书者可能不同。

- **可扩展性**。由于负责特定链码的背书节点与排序节点是无关的，因此系统的 *扩展性* 会比由相同的节点完成这些功能更好。特别是当不同的链码指定不同的背书节点时，这会让背书节点的链码互相隔离，并允许并行执行链码（背书）。因此，代价高昂的链码执行从排序服务的关键路径中删除了。

- **机密性**。本架构有助于部署对其交易的内容和状态更新具有“机密性”要求的链码。

- **共识模块化**。该架构是 *模块化的*，允许可插拔的共识算法（即排序服务）实现。

**第一部分：与 Hyperledger Fabric v1相关的架构元素**

1. 系统架构
2. 交易背书的基本流程
3. 背书策略

**第二部分：v1之后架构版本的元素**

4. 账本检查点(裁剪)

1. 系统架构
----------------------

区块链是一个分布式系统，由许多相互通信的节点组成。区块链运行链码来保存状态和账本数据，并执行交易。链码是核心元素，因为交易是调用链码的操作。交易必须“背书”，只有背书的交易才可以提交并对状态产生影响。可能存在一个或多个用于管理方法和参数的特殊链码，称之为 *系统链码*。

1.1. 交易
~~~~~~~~~~~~~~~~~

交易可以分为两类：

-  *部署交易*，创建新的链码并将程序作为参数。当部署交易成功执行时，链码会被安装在区块链上。

-  *调用交易*，在部署的链码环境中执行操作。调用交易引用链码和它的方法。当成功时，链码执行指定的方法（这可能涉及修改相应的状态），并返回输出。

后边会讲到，部署交易是调用交易的特殊情况，创建新链码的部署交易是系统链码上的调用交易。

**备注**：*本文档目前假设交易要么创建新的链码，要么调用一个已部署链码提供的操作。这个文档还没有描述：a)查询(只读)交易的优化(包含在v1中)，b)对跨链码交易的支持(v1之后版本的特性)*。

1.2. 区块链数据结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1.2.1. 状态
^^^^^^^^^^^^

区块链的最新状态（简称 *状态*）被建模为一个带有版本的键值存储（KVS），其中键是名称，值是任意的。这些条目由区块链上运行的链码（应用程序）通过 ``put`` 和 ``get`` KVS 的方式维护。状态被持久地存储，并记录对状态的更新。注意，带有版本的 KVS 被用作状态模型，实现可以使用已有的 KVS，也可以使用 RDBMS 或任何其他解决方案。

更正式地说，状态 ``s`` 被映射为 ``K -> (V X N)`` 的一个元素，其中:

- ``K`` 是键的集合
- ``V`` 是值的集合
- ``N`` 是一个带有版本号的有序集合。单映射函数 ``next: N -> N`` 接收 ``N`` 并返回下一个版本号。

``V`` 和 ``N`` 都包含一个特殊的元素 |falsum| （空类型），以防 ``N`` 是最小值。初始时所有键都映射为(|falsum|, |falsum|)。对于 ``s(k)=(v,ver)`` ，我们用 ``s(k).value`` 表示 ``v``， 用 ``s(k).version`` 表示 ``ver``。

.. |falsum| unicode:: U+22A5
.. |in| unicode:: U+2208

KVS 操作模型如下：

- ``put(k,v)`` 其中 ``k`` |in| ``K``，``v`` |in| ``V``，获取区块链状态 ``s`` 并改变为 ``s'`` 就是 ``s'(k)=(v,next(s(k).version))``，当 ``k'!=k`` 时 ``s'(k')=s(k')``。
-  ``get(k)`` 返回 ``s(k)``。

状态由 Peer 节点维护，而不是排序节点和客户端。

**状态隔离**。KVS 中的键可以从它们的名称中识别出属于某个特定链码，因为只有特定链码的交易才可以修改属于这个链码的键。原则上，任何链码都可以读取属于其他链码的键。*支持跨链码交易，可以修改属于两个或多个链码的状态，这是一个v1后续版本的特性。*

1.2.2 账本
^^^^^^^^^^^^

账本提供了一个可验证的历史记录，记录了在系统运行期间发生的所有成功的（*有效的*）和失败的（*无效的*）状态更改。

账本是由排序服务（参见第1.3.3节）构造的，它是（有效或无效的）交易的 *区块* 的完全有序哈希链。哈希链强制账本中的区块的顺序，每个区块包含一个完全有序的交易数组。这强制为所有交易指定了顺序。

账本保存在所有的 Peer 节点上，也可以选择保存在部分排序节点上。在排序节点的上下文中，我们将账本称为“排序节点账本”，而在 Peer 节点的上下文中，我们将账本称为“节点账本”。``Peer 节点账本`` 与 ``排序节点账本`` 的不同之处在于，Peer 节点在本地维护一个比特掩码，该比特掩码将有效的交易与无效的交易区分开来（有关详细信息，请参阅XX节）。

如第 XX 节（v1后续版本特性）所述的，Peer 节点可以裁剪 ``节点账本``。排序节点维护“排序节点账本”以获得容错性和（“Peer 节点账本”的）可用性，并可以决定随时对其进行裁剪，前提是排序服务的属性得到了维护（参见第1.3.3节）。

账本允许节点重放所有交易的历史并重建状态。因此，1.2.1节中描述的状态是一个可选的数据结构。

1.3. 节点
~~~~~~~~~~

节点是区块链的通信实体。“节点”只是一个逻辑功能，因为不同类型的多个节点可以运行在同一个物理服务器上。重要的是节点如何在“信任域”中分组并与控制它们的逻辑实体相关联。

有三种类型的节点：

1. **客户端** 或 **提交客户端**：是一个向背书节点提交实际交易调用并向排序服务广播交易提案的客户端。

2. **Peer 节点**：是一个提交交易并维护状态和账本（参见第1.2节）副本的节点。此外，Peer 节点可以扮演一个特殊的 **背书人** 角色。

3. **排序服务节点** 或 **排序节点**：是一个运行实现交付担保的通信服务节点，例如原子性或总顺序广播。

接下来将更详细地解释节点的类型。

1.3.1. 客户端
^^^^^^^^^^^^^

客户端扮演了代表最终用户的实体。它必须连接到与区块链通信的 Peer 节点。客户端可以连接到它所选择的任何 Peer 节点。客户端创建并调用交易。

如第2节所详细介绍的，客户端同时与 Peer 节点和排序服务通信。

1.3.2. Peer 节点
^^^^^^^^^^^^^^^^

Peer 节点接收来自排序服务 *区块* 形式的有序状态更新，并维护状态和账本。

Peer 节点还可以承担 **背书节点** 或 **背书人** 的特殊角色。*背书节点* 发生在一个特定的链码上，包括在提交事务之前 *背书* 交易。每个链码都可以指定一个 *背书策略*，该策略可以引用一组背书节点。策略为有效的交易背书定义了必要条件和充分条件（通常是一组背书人的签名），后面的第2和3节将对此进行讲解。在安装新链码的特殊部署交易的情况下，背书策略指定为系统链码的背书策略。

1.3.3. 排序服务节点（排序节点）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*排序节点* 来自于 *排序服务* ，即提供交付担保的通信结构。排序服务可以以不同的方式实现：从集中式服务（例如，在开发和测试中使用）到针对不同网络和节点故障模型的分布式协议。

排序服务为客户端和 Peer 节点提供共享的 *通信通道*，为包含交易的消息提供广播服务。客户端连接到通道，并可以在通道上向所有 Peer 节点广播消息。该通道支持所有消息以 *原子* 方式传递，即消息通信是全顺序传递和（特定实现）可靠的。换句话说，通道将消息以相同的逻辑顺序输出给所有与之相连的 Peer 节点。这种原子通信保证在分布式系统中也称为 *全顺序广播*、*原子广播* 或 *共识*。所通信的消息是要包含在区块链状态中的候选交易。

**分区(排序服务通道)**。排序服务可能支持多个 *通道*，类似于发布者-订阅者消息系统的 *主题*。客户端可以连接到给定的通道，然后可以发送消息并获取到达的消息。通道可以看作是分区，连接到一个通道的客户端不知道其他通道的存在，但是客户端可以连接到多个通道。尽管 Hyperledger Fabric 中实现的一些排序服务支持多个通道，但为了简化表示，在本文档的其余部分中，我们假设排序服务由一个通道（主题）组成。

**排序服务 API**。Peer 节点通过排序服务提供的接口连接到通道。排序服务 API 由两个基本操作（通常称 *异步事件*）组成：

**TODO** 添加用于在客户端或 Peer 节点获取指定序列号的区块的 API 部分，。

- ``broadcast(blob)``：客户端调用它来在通道上广播任意消息 ``blob``。在 BFT 中，当向服务发送请求时，也称为 ``request(blob)``。

- ``deliver(seqno, prevhash, blob)``：排序服务调用这个来向 Peer 节点发送消息 ``blob``，该消息中包含非负整数序列号（``seqno``）和上一个发送的 ``blob`` 的哈希（``prevhash``）。欢句话说，它是排序服务的输出事件。``deliver()`` 在发布者-订阅者系统中称为 ``notify()``，在 BFT 系统中称为 ``commit()``。

**账本和区块格式**。账本（参见第1.2.2节）中包含了所有排序服务输出的数据。简单来说，它是一个 ``deliver(seqno, prevhash, blob)`` 事件的序列，而事件就是根据前面所说的 ``prevhash`` 计算的哈希链。

大多数时候，出于效率的考虑，排序服务不会输出单个交易（blob），而是在单个 ``deliver`` 事件中将交易分组并输出到 *区块* 中。在这种情况下，排序服务必须限定每个区块中交易的排序。区块中的交易数可以由排序服务动态选择。

为了便于讲解，下边我们定义了排序服务属性（本节剩余部分）并解释了交易背书工作流（第二节），其中我们假设每个 ``deliver`` 事件中只有一个交易。这些很容易扩展到区块上，根据上面提到的区块中交易的确定性顺序，假设一个区块的 ``deliver`` 事件对应于一个区块中的每个交易单独的 ``deliver`` 事件序列。

**排序服务属性**

排序服务（或原子广播通道）的保证规定了广播了什么消息，以及传递的消息之间存在什么关系。这些保证如下:

1. **安全（一致性保证）**：只要 Peer 节点连接到通道的时间足够长（它们可以断开连接或崩溃，但会重新启动和重新连接），它们将看到一个 *相同的* 已交付的 ``(seqno, prevhash, blob)`` 消息序列。这意味着所有 Peer 节点都可以收到 *相同顺序* 的输出（``deliver()`` 事件），并且相同的序列号都有 *相同的内容* （``blob`` 和 ``prevhash``）。注意，这只是一个 *逻辑顺序*，一个 Peer 节点上的``deliver(seqno, prevhash, blob)`` 不需要与另一个 Peer 节点上输出相同的 ``deliver(seqno, prevhash, blob)`` 的消息发生实时关联。换句话说，给定一个特定的 ``seqno``，*没有* 两个正确的 Peer 节点会提供 *不同的* ``prevhash`` 或 ``blob`` 值。此外，除非某个客户端（Peer 节点）实际调用了 ``broadcast(blob)``，否则不会传递任何 ``blob``，即每个广播过的 blob 只分发 *一次*。

   此外，``deliver()`` 事件包含前一个 ``deliver()`` 事件中的数据的哈希（``prevhash``）。当排序服务实现原子广播保证时，``prevhash`` 是 ``deliver()`` 事件的参数和序号 ``seqno-1`` 的哈希。这将在 ``deliver()`` 事件之间建立一个哈希链，用于帮助验证排序服务输出的完整性，第4和第5节将讨论这个。在第一个 ``deliver()`` 事件中 ``prevhash`` 有一个默认值。

2. **存活性(交付保证)**：排序服务的存活性保证由排序服务决定。准确的保证取决于网络和节点故障模型。

   原则上，如果提交的客户端没有失败，那么排序服务应该确保连接到排序服务的每个正确的 Peer 节点最终会发送每个提交的交易。

总而言之，排序服务确保以下特性：

- *协议*。对于任何两个正确的 Peer 节点上有相同 ``seqno`` 的事件 ``deliver(seqno, prevhash0, blob0)`` 和 ``deliver(seqno, prevhash1, blob1)``，``prevhash0==prevhash1`` 并且 ``blob0==blob1``；

- *哈希链完整性*。对于任何两个正确的 Peer 节点上的事件 ``deliver(seqno, prevhash0, blob0)`` 和 ``deliver(seqno, prevhash1, blob1)``，``prevhash = HASH(seqno-1||prevhash0||blob0)``

- *不能跳跃*。如果排序服务向正确的 Peer 节点 *p* 输出了 ``deliver(seqno, prevhash, blob)``，其中 ``seqno>0``。那么 *p* 肯定已经接收到了 ``deliver(seqno-1, prevhash0, blob0)``。

- *不能创建*。正确的Peer 节点上的 ``deliver(seqno, prevhash, blob)`` 事件必须在一些（可能不是同一个） Peer 节点上通过 ``broadcast(blob)`` 事件处理。

- *没有重复（可选，但最好存在）*。对于两个事件 ``broadcast(blob)`` 和 ``broadcast(blob')`` ，当两个事件 ``deliver(seqno0, prevhash0, blob)`` 和 ``deliver(seqno1, prevhash1, blob')`` 在正确的 Peer 上发生时，并且 ``blob == blob'``，那么 ``seqno0==seqno1`` 而且 ``prevhash0==prevhash1``。

- *存活性*。如果一个正确的客户端调用一个事件 ``broadcast(blob)``，那么每个正确的 Peer 节点“最终”都会发出一个事件 ``deliver(*, *, blob)``，其中 ``*`` 表示一个任意值。

2. 交易背书的基本流程
--------------------------------------------

在下面的文章中，我们将概述交易请求的整体流程。

**注：** *注意以下协议并不假设所有交易都是确定性的，即它允许非确定性交易。*

2.1. 客户端创建一个交易并将其发送给它所指定的背书节点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

要执行一个交易，客户端需要向其指定的背书节点发送一个 ``提案``（PROPOSE）消息（可能不是同时。请参见2.1.2和2.3节）。客户端可以通过 Peer 节点根据背书策略（参阅第三章）得到给定 ``chaincodeID`` 的背书节点。例如，交易可以发送给指定 ``chaincodeID`` 的 *所有* 背书节点。也就是说，一些背书节点可以离线，其他的可能不同意或者不背书该交易。提交客户端可以尝试可用的背书节点来满足背书策略。

下边我们将首先详细介绍 ``提案`` 消息格式，然后讨论提交客户端和背书节点之间可能的交互模式。

2.1.1. ``提案`` 消息格式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``提案`` 消息的格式是 ``<PROPOSE,tx,[anchor]>``，``tx`` 是必选参数而 ``anchor`` 是可选参数，解释如下。

- ``tx=<clientID,chaincodeID,txPayload,timestamp,clientSig>``，其中
  - ``clientID`` 是提交客户端的 ID，
  - ``chaincodeID`` 是提交的交易所引用的链码，
  - ``txPayload`` 是提交的交易所包含的内容，
  - ``timestamp`` 是由客户端维护的单调递增（对于每一个新交易）的整数，
  - ``clientSig`` 是客户端对 ``tx`` 其他字段的签名。

  在执行交易和部署交易中 ``txPayload`` 的细节所有不同，对于 **执行交易**，``txPayload`` 包含两个字段：
  
  - ``txPayload = <operation, metadata>``，其中
    - ``operation`` 定义了链码方法和参数，
    - ``metadata`` 定义了执行相关的参数。

  对于 **部署交易**,``txPayload`` 包含三个字段：
  -  ``txPayload = <source, metadata, policies>``， 其中

      -  ``source`` 定义了链码的源码，
      -  ``metadata`` 定义了相关的链码和应用程序，
      -  ``policies`` 包含了链码相关的策略，比如背书策略，它可以被所有 Peer 节点访问。
         注意，在 ``部署`` 交易中 ``txPayload`` 不包含背书策略，但是包含背书策略 ID 和它的参数（参见第三章）。

- ``anchor`` 包含了 *读版本依赖项*，具体来说就是“键值-版本”对（即 ``anchor`` 是 ``KxN`` 的子集），它将 ``提案`` 请求绑定或者“锚定”在 KVS（参见 1.2）中指定的键的版本上。如果客户端指定了 ``anchor`` 参数，背书节点仅在其本地 KVS 和 ``anchor`` 对应键的 *读* 版本号相匹配时才背书交易（更多的细节参见第2.2节）。

所有节点都是用 ``tx`` 的哈希作为交易标识符 ``tid``，即 ``tid=HASH(tx)``。客户端将 ``tid`` 保存在内存中，等待背书节点的响应。

2.1.2. 消息模式
^^^^^^^^^^^^^^^^^^^^^^^

The client decides on the sequence of interaction with endorsers. For
example, a client would typically send ``<PROPOSE, tx>`` (i.e., without
the ``anchor`` argument) to a single endorser, which would then produce
the version dependencies (``anchor``) which the client can later on use
as an argument of its ``PROPOSE`` message to other endorsers. As another
example, the client could directly send ``<PROPOSE, tx>`` (without
``anchor``) to all endorsers of its choice. Different patterns of
communication are possible and client is free to decide on those (see
also Section 2.3.).

2.2. The endorsing peer simulates a transaction and produces an endorsement signature
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On reception of a ``<PROPOSE,tx,[anchor]>`` message from a client, the
endorsing peer ``epID`` first verifies the client's signature
``clientSig`` and then simulates a transaction. If the client specifies
``anchor`` then endorsing peer simulates the transactions only upon read
version numbers (i.e., ``readset`` as defined below) of corresponding
keys in its local KVS match those version numbers specified by
``anchor``.

Simulating a transaction involves endorsing peer tentatively *executing*
a transaction (``txPayload``), by invoking the chaincode to which the
transaction refers (``chaincodeID``) and the copy of the state that the
endorsing peer locally holds.

As a result of the execution, the endorsing peer computes *read version
dependencies* (``readset``) and *state updates* (``writeset``), also
called *MVCC+postimage info* in DB language.

Recall that the state consists of key-value pairs. All key-value entries
are versioned; that is, every entry contains ordered version
information, which is incremented each time the value stored under
a key is updated. The peer that interprets the transaction records all
key-value pairs accessed by the chaincode, either for reading or for writing,
but the peer does not yet update its state. More specifically:

-  Given state ``s`` before an endorsing peer executes a transaction,
   for every key ``k`` read by the transaction, pair
   ``(k,s(k).version)`` is added to ``readset``.
-  Additionally, for every key ``k`` modified by the transaction to the
   new value ``v'``, pair ``(k,v')`` is added to ``writeset``.
   Alternatively, ``v'`` could be the delta of the new value to previous
   value (``s(k).value``).

If a client specifies ``anchor`` in the ``PROPOSE`` message then client
specified ``anchor`` must equal ``readset`` produced by endorsing peer
when simulating the transaction.

Then, the peer forwards internally ``tran-proposal`` (and possibly
``tx``) to the part of its (peer's) logic that endorses a transaction,
referred to as **endorsing logic**. By default, endorsing logic at a
peer accepts the ``tran-proposal`` and simply signs the
``tran-proposal``. However, endorsing logic may interpret arbitrary
functionality, to, e.g., interact with legacy systems with
``tran-proposal`` and ``tx`` as inputs to reach the decision whether to
endorse a transaction or not.

If endorsing logic decides to endorse a transaction, it sends
``<TRANSACTION-ENDORSED, tid, tran-proposal,epSig>`` message to the
submitting client(\ ``tx.clientID``), where:

-  ``tran-proposal := (epID,tid,chaincodeID,txContentBlob,readset,writeset)``,

   where ``txContentBlob`` is chaincode/transaction specific
   information. The intention is to have ``txContentBlob`` used as some
   representation of ``tx`` (e.g., ``txContentBlob=tx.txPayload``).

-  ``epSig`` is the endorsing peer's signature on ``tran-proposal``

Else, in case the endorsing logic refuses to endorse the transaction, an
endorser *may* send a message ``(TRANSACTION-INVALID, tid, REJECTED)``
to the submitting client.

Notice that an endorser does not change its state in this step, the
updates produced by transaction simulation in the context of endorsement
do not affect the state!

2.3. The submitting client collects an endorsement for a transaction and broadcasts it through ordering service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The submitting client waits until it receives "enough" messages and
signatures on ``(TRANSACTION-ENDORSED, tid, *, *)`` statements to
conclude that the transaction proposal is endorsed. As discussed in
Section 2.1.2., this may involve one or more round-trips of interaction
with endorsers.

The exact number of "enough" depend on the chaincode endorsement policy
(see also Section 3). If the endorsement policy is satisfied, the
transaction has been *endorsed*; note that it is not yet committed. The
collection of signed ``TRANSACTION-ENDORSED`` messages from endorsing
peers which establish that a transaction is endorsed is called an
*endorsement* and denoted by ``endorsement``.

If the submitting client does not manage to collect an endorsement for a
transaction proposal, it abandons this transaction with an option to
retry later.

For transaction with a valid endorsement, we now start using the
ordering service. The submitting client invokes ordering service using
the ``broadcast(blob)``, where ``blob=endorsement``. If the client does
not have capability of invoking ordering service directly, it may proxy
its broadcast through some peer of its choice. Such a peer must be
trusted by the client not to remove any message from the ``endorsement``
or otherwise the transaction may be deemed invalid. Notice that,
however, a proxy peer may not fabricate a valid ``endorsement``.

2.4. The ordering service delivers a transactions to the peers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When an event ``deliver(seqno, prevhash, blob)`` occurs and a peer has
applied all state updates for blobs with sequence number lower than
``seqno``, a peer does the following:

-  It checks that the ``blob.endorsement`` is valid according to the
   policy of the chaincode (``blob.tran-proposal.chaincodeID``) to which
   it refers.

-  In a typical case, it also verifies that the dependencies
   (``blob.endorsement.tran-proposal.readset``) have not been violated
   meanwhile. In more complex use cases, ``tran-proposal`` fields in
   endorsement may differ and in this case endorsement policy (Section
   3) specifies how the state evolves.

Verification of dependencies can be implemented in different ways,
according to a consistency property or "isolation guarantee" that is
chosen for the state updates. **Serializability** is a default isolation
guarantee, unless chaincode endorsement policy specifies a different
one. Serializability can be provided by requiring the version associated
with *every* key in the ``readset`` to be equal to that key's version in
the state, and rejecting transactions that do not satisfy this
requirement.

-  If all these checks pass, the transaction is deemed *valid* or
   *committed*. In this case, the peer marks the transaction with 1 in
   the bitmask of the ``PeerLedger``, applies
   ``blob.endorsement.tran-proposal.writeset`` to blockchain state (if
   ``tran-proposals`` are the same, otherwise endorsement policy logic
   defines the function that takes ``blob.endorsement``).

-  If the endorsement policy verification of ``blob.endorsement`` fails,
   the transaction is invalid and the peer marks the transaction with 0
   in the bitmask of the ``PeerLedger``. It is important to note that
   invalid transactions do not change the state.

Note that this is sufficient to have all (correct) peers have the same
state after processing a deliver event (block) with a given sequence
number. Namely, by the guarantees of the ordering service, all correct
peers will receive an identical sequence of
``deliver(seqno, prevhash, blob)`` events. As the evaluation of the
endorsement policy and evaluation of version dependencies in ``readset``
are deterministic, all correct peers will also come to the same
conclusion whether a transaction contained in a blob is valid. Hence,
all peers commit and apply the same sequence of transactions and update
their state in the same way.

.. _swimlane:

.. image:: images/flow-4.png
   :alt: Illustration of the transaction flow (common-case path).

*Figure 1. Illustration of one possible transaction flow (common-case path).*

3. Endorsement policies
-----------------------

3.1. Endorsement policy specification
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An **endorsement policy**, is a condition on what *endorses* a
transaction. Blockchain peers have a pre-specified set of endorsement
policies, which are referenced by a ``deploy`` transaction that installs
specific chaincode. Endorsement policies can be parametrized, and these
parameters can be specified by a ``deploy`` transaction.

To guarantee blockchain and security properties, the set of endorsement
policies **should be a set of proven policies** with limited set of
functions in order to ensure bounded execution time (termination),
determinism, performance and security guarantees.

Dynamic addition of endorsement policies (e.g., by ``deploy``
transaction on chaincode deploy time) is very sensitive in terms of
bounded policy evaluation time (termination), determinism, performance
and security guarantees. Therefore, dynamic addition of endorsement
policies is not allowed, but can be supported in future.

3.2. Transaction evaluation against endorsement policy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A transaction is declared valid only if it has been endorsed according
to the policy. An invoke transaction for a chaincode will first have to
obtain an *endorsement* that satisfies the chaincode's policy or it will
not be committed. This takes place through the interaction between the
submitting client and endorsing peers as explained in Section 2.

Formally the endorsement policy is a predicate on the endorsement, and
potentially further state that evaluates to TRUE or FALSE. For deploy
transactions the endorsement is obtained according to a system-wide
policy (for example, from the system chaincode).

An endorsement policy predicate refers to certain variables. Potentially
it may refer to:

1. keys or identities relating to the chaincode (found in the metadata
   of the chaincode), for example, a set of endorsers;
2. further metadata of the chaincode;
3. elements of the ``endorsement`` and ``endorsement.tran-proposal``;
4. and potentially more.

The above list is ordered by increasing expressiveness and complexity,
that is, it will be relatively simple to support policies that only
refer to keys and identities of nodes.

**The evaluation of an endorsement policy predicate must be
deterministic.** An endorsement shall be evaluated locally by every peer
such that a peer does *not* need to interact with other peers, yet all
correct peers evaluate the endorsement policy in the same way.

3.3. Example endorsement policies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The predicate may contain logical expressions and evaluates to TRUE or
FALSE. Typically the condition will use digital signatures on the
transaction invocation issued by endorsing peers for the chaincode.

Suppose the chaincode specifies the endorser set
``E = {Alice, Bob, Charlie, Dave, Eve, Frank, George}``. Some example
policies:

-  A valid signature from on the same ``tran-proposal`` from all members
   of E.

-  A valid signature from any single member of E.

-  Valid signatures on the same ``tran-proposal`` from endorsing peers
   according to the condition
   ``(Alice OR Bob) AND (any two of: Charlie, Dave, Eve, Frank, George)``.

-  Valid signatures on the same ``tran-proposal`` by any 5 out of the 7
   endorsers. (More generally, for chaincode with ``n > 3f`` endorsers,
   valid signatures by any ``2f+1`` out of the ``n`` endorsers, or by
   any group of *more* than ``(n+f)/2`` endorsers.)

-  Suppose there is an assignment of "stake" or "weights" to the
   endorsers, like
   ``{Alice=49, Bob=15, Charlie=15, Dave=10, Eve=7, Frank=3, George=1}``,
   where the total stake is 100: The policy requires valid signatures
   from a set that has a majority of the stake (i.e., a group with
   combined stake strictly more than 50), such as ``{Alice, X}`` with
   any ``X`` different from George, or
   ``{everyone together except Alice}``. And so on.

-  The assignment of stake in the previous example condition could be
   static (fixed in the metadata of the chaincode) or dynamic (e.g.,
   dependent on the state of the chaincode and be modified during the
   execution).

-  Valid signatures from (Alice OR Bob) on ``tran-proposal1`` and valid
   signatures from ``(any two of: Charlie, Dave, Eve, Frank, George)``
   on ``tran-proposal2``, where ``tran-proposal1`` and
   ``tran-proposal2`` differ only in their endorsing peers and state
   updates.

How useful these policies are will depend on the application, on the
desired resilience of the solution against failures or misbehavior of
endorsers, and on various other properties.

4 (post-v1). Validated ledger and ``PeerLedger`` checkpointing (pruning)
------------------------------------------------------------------------

4.1. Validated ledger (VLedger)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To maintain the abstraction of a ledger that contains only valid and
committed transactions (that appears in Bitcoin, for example), peers
may, in addition to state and Ledger, maintain the *Validated Ledger (or
VLedger)*. This is a hash chain derived from the ledger by filtering out
invalid transactions.

The construction of the VLedger blocks (called here *vBlocks*) proceeds
as follows. As the ``PeerLedger`` blocks may contain invalid
transactions (i.e., transactions with invalid endorsement or with
invalid version dependencies), such transactions are filtered out by
peers before a transaction from a block becomes added to a vBlock. Every
peer does this by itself (e.g., by using the bitmask associated with
``PeerLedger``). A vBlock is defined as a block without the invalid
transactions, that have been filtered out. Such vBlocks are inherently
dynamic in size and may be empty. An illustration of vBlock construction
is given in the figure below.

.. image:: images/blocks-3.png
   :alt: Illustration of vBlock formation

*Figure 2. Illustration of validated ledger block (vBlock) formation from ledger (PeerLedger) blocks.*

vBlocks are chained together to a hash chain by every peer. More
specifically, every block of a validated ledger contains:

-  The hash of the previous vBlock.

-  vBlock number.

-  An ordered list of all valid transactions committed by the peers
   since the last vBlock was computed (i.e., list of valid transactions
   in a corresponding block).

-  The hash of the corresponding block (in ``PeerLedger``) from which
   the current vBlock is derived.

All this information is concatenated and hashed by a peer, producing the
hash of the vBlock in the validated ledger.

4.2. ``PeerLedger`` Checkpointing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ledger contains invalid transactions, which may not necessarily be
recorded forever. However, peers cannot simply discard ``PeerLedger``
blocks and thereby prune ``PeerLedger`` once they establish the
corresponding vBlocks. Namely, in this case, if a new peer joins the
network, other peers could not transfer the discarded blocks (pertaining
to ``PeerLedger``) to the joining peer, nor convince the joining peer of
the validity of their vBlocks.

To facilitate pruning of the ``PeerLedger``, this document describes a
*checkpointing* mechanism. This mechanism establishes the validity of
the vBlocks across the peer network and allows checkpointed vBlocks to
replace the discarded ``PeerLedger`` blocks. This, in turn, reduces
storage space, as there is no need to store invalid transactions. It
also reduces the work to reconstruct the state for new peers that join
the network (as they do not need to establish validity of individual
transactions when reconstructing the state by replaying ``PeerLedger``,
but may simply replay the state updates contained in the validated
ledger).

4.2.1. Checkpointing protocol
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Checkpointing is performed periodically by the peers every *CHK* blocks,
where *CHK* is a configurable parameter. To initiate a checkpoint, the
peers broadcast (e.g., gossip) to other peers message
``<CHECKPOINT,blocknohash,blockno,stateHash,peerSig>``, where
``blockno`` is the current blocknumber and ``blocknohash`` is its
respective hash, ``stateHash`` is the hash of the latest state (produced
by e.g., a Merkle hash) upon validation of block ``blockno`` and
``peerSig`` is peer's signature on
``(CHECKPOINT,blocknohash,blockno,stateHash)``, referring to the
validated ledger.

A peer collects ``CHECKPOINT`` messages until it obtains enough
correctly signed messages with matching ``blockno``, ``blocknohash`` and
``stateHash`` to establish a *valid checkpoint* (see Section 4.2.2.).

Upon establishing a valid checkpoint for block number ``blockno`` with
``blocknohash``, a peer:

-  if ``blockno>latestValidCheckpoint.blockno``, then a peer assigns
   ``latestValidCheckpoint=(blocknohash,blockno)``,
-  stores the set of respective peer signatures that constitute a valid
   checkpoint into the set ``latestValidCheckpointProof``,
-  stores the state corresponding to ``stateHash`` to
   ``latestValidCheckpointedState``,
-  (optionally) prunes its ``PeerLedger`` up to block number ``blockno``
   (inclusive).

4.2.2. Valid checkpoints
^^^^^^^^^^^^^^^^^^^^^^^^

Clearly, the checkpointing protocol raises the following questions:
*When can a peer prune its ``PeerLedger``? How many ``CHECKPOINT``
messages are "sufficiently many"?*. This is defined by a *checkpoint
validity policy*, with (at least) two possible approaches, which may
also be combined:

-  *Local (peer-specific) checkpoint validity policy (LCVP).* A local
   policy at a given peer *p* may specify a set of peers which peer *p*
   trusts and whose ``CHECKPOINT`` messages are sufficient to establish
   a valid checkpoint. For example, LCVP at peer *Alice* may define that
   *Alice* needs to receive ``CHECKPOINT`` message from Bob, or from
   *both* *Charlie* and *Dave*.

-  *Global checkpoint validity policy (GCVP).* A checkpoint validity
   policy may be specified globally. This is similar to a local peer
   policy, except that it is stipulated at the system (blockchain)
   granularity, rather than peer granularity. For instance, GCVP may
   specify that:

   -  each peer may trust a checkpoint if confirmed by *11* different
      peers.
   -  in a specific deployment in which every orderer is collocated with
      a peer in the same machine (i.e., trust domain) and where up to
      *f* orderers may be (Byzantine) faulty, each peer may trust a
      checkpoint if confirmed by *f+1* different peers collocated with
      orderers.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
