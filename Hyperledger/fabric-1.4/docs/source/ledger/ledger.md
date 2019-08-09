# 账本

**受众**：架构师、应用程序开发者和智能合约开发者、管理员

**账本**是 Hyperledger Fabric 中的一个重要概念，它存储有关业务对象的重要信息，包括对象属性的当前值和产生这些当前值的交易的历史。

在这个主题中，我们将涉及：

* [什么是账本?](#what-is-a-ledger?)
* [业务对象的实际存储](#ledgers-facts-and-states)
* [区块链账本](#a-blockchain-ledger)
* [世界状态](#world-state)
* [区块链数据结构](#blockchain)
* [区块链如何存储区块](#blocks)
* [交易](#transactions)
* [世界状态数据库选择](#world-state-database-options)
* [Fabcar 示例账本](#example-ledger-fabcar)
* [账本和命名空间](#namespaces)
* [账本和通道](#channels)

## 什么是账本？

账本包含业务的当前状态，就像一个交易日记。欧洲和中国最早的账本可以追溯到近 1000 年前，苏美尔人在 4000 年前就有了[石制账本](http://www.sciencephoto.com/media/686227/view/accounting-ledger-sumerian-cuneiform)，但让我们从一个更现代的例子开始吧！

你可能已经习惯查看你的银行账户了。对你来说，最重要的是可用的余额，它是你现在能花多少钱。如果你想看看你的余额是如何产生出来的，那么你可以查看收入和支出的交易记录。这是现实生活中的一个账本的示例——一个状态（您的银行余额）和一组确定账本的有序交易（收入和支出）。Hyperledger Fabric 同样是基于这两个问题而产生的，它显示一组账本状态的当前值，以及记录决定这些状态的交易历史。

## 账本、事实和状态

账本并不真正地存储业务对象，而是存储关于这些对象的**事实**。当我们说“我们在账本中存储一个业务对象”时，我们真正的意思是我们正在记录关于一个对象当前状态的事实，以及关于导致当前状态的交易历史的事实。在一个日益数字化的世界里，我们感觉自己在看一个物体，而不是关于一个物体的事实。对于数字对象，它很可能存在于外部数据存储中；我们存储在账本中的事实使我们能够确定它的位置以及有关它的其他关键信息。

虽然关于业务对象当前状态的事实可能会改变，但是关于它的事实的历史是**不可变的**，可以将其添加到其中，但不能对其进行回溯性更改。我们将看到，将区块链看作业务对象事实的不可变历史，是理解它的一种简单而有效的方法。

现在让我们仔细看看 Hyperledger Fabric 的账本结构！

## 账本

在 Hyperledger Fabric 中，账本由两个不同但相关的部分组成：一个世界状态和一个区块链。每一个都表示一组关于一组业务对象的事实。

首先，有一个**世界状态**，它是一个数据库，包含了一组账本状态的**当前值**的缓存。世界状态使程序可以很容易地直接访问状态的当前值，而不必遍历整个交易日志来计算它。默认情况下，账本状态表示为**键值对**，稍后我们将看到 Hyperledger Fabric 如何在这方面提供灵活性。世界状态可以频繁地更改，因为状态可以被创建、更新和删除。

其次，还有一个**区块链**，它记录改变当前世界状态的所有交易日志。交易收集在附加到区块链的区块中，以便了解导致当前世界状态的更改的历史。区块链数据结构与世界状态非常不同，因为一旦写入，就无法修改，它是**不可篡改的**。

![ledger.ledger](./ledger.diagram.1.png) 

*账本 L 包括区块链 B 和世界状态 W，世界状态 W 由区块链 B 决定。我们也可以说世界状态 W 继承自区块链 B。*

在一个 Hyperledger Fabric 网络中**逻辑**账本很有用。实际上，该网络通过**共识**维护一个账本的多个副本，这些副本与其他副本保持一致。**分布式账本技术**（DLT）这个术语经常与这种账本联系在一起，这种账本在逻辑上是单一的，但是在整个网络中分布着许多一致的副本。

现在让我们更详细地研究世界状态和区块链数据结构。

## 世界状态

世界状态将业务对象属性的当前值保存为唯一的账本状态。这很有用，因为程序通常需要对象的当前值，遍历整个区块链来计算对象的当前值会很麻烦，您可以直接从世界状态获取它。

![ledger.worldstate](./ledger.diagram.3.png) 

*一个账本世界状态包含两个状态。第一个状态是： key=CAR1 和 value=Audi。第二个状态有一个更复杂的值：key=CAR2 和 value={model:BMW, color=red, owner=Jane} 。两个状态的版本都是0。*

账本状态记录一组关于特定业务对象的事实。我们的示例显示了 CAR1 和 CAR2 这两辆车的账本状态，每辆车都有一个键和一个值。应用程序可以调用智能合约，该合约使用简单的账本 API 来**获取**、**设置**和**删除**状态。注意状态值可以简单值（Audi...），也可以是复合值（type:BMW...）。通常通过查询世界状态来检索具有特定属性的对象，例如查找所有红色宝马。

世界状态用数据库实现。这很有意义，因为数据库提供了一组丰富的操作符来有效地存储和检索状态。稍后我们将看到，可以将 Hyperledger Fabric 配置为使用不同的世界状态数据库来满足不同类型的状态值和应用程序所需的访问模式的需要，例如，复杂查询。

应用程序提交更改世界状态的交易，这些交易最终提交到账本区块链。应用程序通过 Hyperledger Fabric SDK 与这种[共识机制](../txflow.html)的细节隔离，它们仅仅调用一个智能合约，当交易被包含在区块链中时（无论是否有效），它们都会被通知。重点是，只有被**背书组织签名**的交易才会导致对世界状态的更新。如果一个交易没有足够的背书者签名，它将不会导致世界状态的改变。您可以阅读更多关于应用程序如何使用[智能合约](../smartcontract/smartcontract.html)以及如何[开发应用程序](../developapps/developing_applications.html)的信息。

您还会注意到，状态有一个版本号，在上面的图表中，状态 CAR1 和 CAR2 处于它们的初始版本 0。版本号用于 Hyperledger Fabric 内部，并在每次状态更改时递增。每当更新状态时，都会检查版本，以确保当前状态与背书时的版本匹配。这就确保了世界状态正在按照预期发生变化，没有被更新。

最后，当第一次创建账本时，世界状态为空。因为任何对世界状态有效更改的交易都记录在区块链上，这意味着可以随时从区块链重新生成世界状态。这非常方便，例如，创建节点时自动生成世界状态。此外，如果某个节点发生异常，则可以在节点重新启动后，接受交易之前，重新生成世界状态。

## 区块链

现在让我们把注意力从世界状态转移到区块链。虽然世界状态包含一组与一组业务对象的当前状态相关的事实，但是区块链是关于这些对象如何达到当前状态的事实的历史记录。区块链记录了每个账本状态的每个历史版本以及它是如何被更改的。

区块链结构为相互链接的区块的顺序日志，其中每个区块包含一系列交易，每个交易表示对世界状态的查询或更新。[其他地方](../peers/peers.html#peers-and-orderers)讨论了交易的确切排序机制，重要的是，区块排序，以及区块内的交易排序，是在称为**排序服务**的 Hyperledger Fabric 组件首次创建区块时建立的。

每个区块的头部都包含区块交易的哈希，以及前一个区块头哈希的副本。这样，账本上的所有交易都按顺序排列，并以密码方式连接在一起。这种哈希和链接使账本数据非常安全。即使一个保存账本的节点被篡改了，它也不能让所有其他节点相信它拥有“正确的”区块链，因为账本分布在一个由独立节点组成的网络中。

与使用数据库的世界状态不同，区块链以文件方式实现。这是一个明智的设计，因为区块链数据结构主要偏向于非常小的一组简单操作。附加到区块链末尾的操作是主要操作，查询目前是一个相对不频繁的操作。

让我们更详细地看看区块链的结构。

![ledger.blockchain](./ledger.diagram.2.png) 

*区块链 B 包含区块 B0、B1、B2、B3。B0 是它的第一个区块，也就是初始区块。*

在上面的图中，我们可以看到**区块** B2 有一个**区块数据** D2，它包含所有的交易：T5、T6、T7。

最重要的是，B2 有一个**区块头** H2，它包含 D2 中所有交易的加密**哈希**，以及与前一个区块 B1 相同的哈希。通过这种方式，区块之间不可分割地、不可改变地链接在一起，术语**区块链**很好地描述了这一点！

最后，如图所示，区块链中的第一个区块称为**初始区块**。它是账本的起点，尽管它不包含任何用户交易。相反，它包含一个配置交易，其中包含网络通道的初始状态（未显示）。在讨论区块链网络和[通道](../channels.html) 时，我们将更详细地讨论初始区块。

## 区块

让我们仔细看看一个区块的结构。它由三个部分组成

* **区块头**

  This section comprises three fields, written when a block is created.

  * **Block number**: An integer starting at 0 (the genesis block), and
  increased by 1 for every new block appended to the blockchain.

  * **Current Block Hash**: The hash of all the transactions contained in the
  current block.

  * **Previous Block Hash**: A copy of the hash from the previous block in the
  blockchain.

  These fields are internally derived by cryptographically hashing the block
  data. They ensure that each and every block is inextricably linked to its
  neighbour, leading to an immutable ledger.

  ![ledger.blocks](./ledger.diagram.4.png) *Block header details. The header H2
  of block B2 consists of block number 2, the hash CH2 of the current block data
  D2, and a copy of a hash PH1 from the previous block, block number 1.*


* **Block Data**

  This section contains a list of transactions arranged in order. It is written
  when the block is created by the ordering service. These transactions have a
  rich but straightforward structure, which we describe [later](#Transactions)
  in this topic.


* **Block Metadata**

  This section contains the time when the block was written, as well as the
  certificate, public key and signature of the block writer. Subsequently, the
  block committer also adds a valid/invalid indicator for every transaction,
  though this information is not included in the hash, as that is created when
  the block is created.

## Transactions

As we've seen, a transaction captures changes to the world state. Let's have a
look at the detailed **blockdata** structure which contains the transactions in
a block.

![ledger.transaction](./ledger.diagram.5.png) *Transaction details. Transaction
T4 in blockdata D1 of block B1 consists of transaction header, H4, a transaction
signature, S4, a transaction proposal P4, a transaction response, R4, and a list
of endorsements, E4.*

In the above example, we can see the following fields:


* **Header**

  This section, illustrated by H4, captures some essential metadata about the
  transaction -- for example, the name of the relevant chaincode, and its
  version.


* **Signature**

  This section, illustrated by S4, contains a cryptographic signature, created
  by the client application. This field is used to check that the transaction
  details have not been tampered with, as it requires the application's private
  key to generate it.


* **Proposal**

  This field, illustrated by P4, encodes the input parameters supplied by an
  application to the smart contract which creates the proposed ledger update.
  When the smart contract runs, this proposal provides a set of input
  parameters, which, in combination with the current world state, determines the
  new world state.


* **Response**

  This section, illustrated by R4, captures the before and after values of the
  world state, as a **Read Write set** (RW-set). It's the output of a smart
  contract, and if the transaction is successfully validated, it will be applied
  to the ledger to update the world state.


* **Endorsements**

  As shown in E4, this is a list of signed transaction responses from each
  required organization sufficient to satisfy the endorsement policy. You'll
  notice that, whereas only one transaction response is included in the
  transaction, there are multiple endorsements. That's because each endorsement
  effectively encodes its organization's particular transaction response --
  meaning that there's no need to include any transaction response that doesn't
  match sufficient endorsements as it will be rejected as invalid, and not
  update the world state.

That concludes the major fields of the transaction -- there are others, but
these are the essential ones that you need to understand to have a solid
understanding of the ledger data structure.

## World State database options

The world state is physically implemented as a database, to provide simple and
efficient storage and retrieval of ledger states. As we've seen, ledger states
can have simple or compound values, and to accommodate this, the world state
database implementation can vary, allowing these values to be efficiently
implemented. Options for the world state database currently include LevelDB and
CouchDB.

LevelDB is the default and is particularly appropriate when ledger states are
simple key-value pairs. A LevelDB database is closely co-located with a network
node -- it is embedded within the same operating system process.

CouchDB is a particularly appropriate choice when ledger states are structured
as JSON documents because CouchDB supports the rich queries and update of richer
data types often found in business transactions. Implementation-wise, CouchDB
runs in a separate operating system process, but there is still a 1:1 relation
between a peer node and a CouchDB instance. All of this is invisible to a smart
contract. See [CouchDB as the StateDatabase](../couchdb_as_state_database.html)
for more information on CouchDB.

In LevelDB and CouchDB, we see an important aspect of Hyperledger Fabric -- it
is *pluggable*. The world state database could be a relational data store, or a
graph store, or a temporal database.  This provides great flexibility in the
types of ledger states that can be efficiently accessed, allowing Hyperledger
Fabric to address many different types of problems.

## Example Ledger: fabcar

As we end this topic on the ledger, let's have a look at a sample ledger. If
you've run the [fabcar sample application](../write_first_app.html), then you've
created this ledger.

The fabcar sample app creates a set of 10 cars each with a unique identity; a
different color, make, model and owner. Here's what the ledger looks like after
the first four cars have been created.

![ledger.transaction](./ledger.diagram.6.png) *The ledger, L, comprises a world
state, W and a blockchain, B. W contains four states with keys: CAR0, CAR2, CAR3
and CAR4. B contains two blocks, 0 and 1. Block 1 contains four transactions:
T1, T2, T3, T4.*

We can see that the world state contains states that correspond to CAR0, CAR1,
CAR2 and CAR3. CAR0 has a value which indicates that it is a blue Toyota Prius,
currently owned by Tomoko, and we can see similar states and values for the
other cars. Moreover, we can see that all car states are at version number 0,
indicating that this is their starting version number -- they have not been
updated since they were created.

We can also see that the blockchain contains two blocks.  Block 0 is the genesis
block, though it does not contain any transactions that relate to cars. Block 1
however, contains transactions T1, T2, T3, T4 and these correspond to
transactions that created the initial states for CAR0 to CAR3 in the world
state. We can see that block 1 is linked to block 0.

We have not shown the other fields in the blocks or transactions, specifically
headers and hashes.  If you're interested in the precise details of these, you
will find a dedicated reference topic elsewhere in the documentation. It gives
you a fully worked example of an entire block with its transactions in glorious
detail -- but for now, you have achieved a solid conceptual understanding of a
Hyperledger Fabric ledger. Well done!

## Namespaces

Even though we have presented the ledger as though it were a single world state
and single blockchain, that's a little bit of an over-simplification. In
reality, each chaincode has its own world state that is separate from all other
chaincodes. World states are in a namespace so that only smart contracts within
the same chaincode can access a given namespace.

A blockchain is not namespaced. It contains transactions from many different
smart contract namespaces. You can read more about chaincode namespaces in this
[topic](./developapps/chaincodenamespace.html).

Let's now look at how the concept of a namespace is applied within a Hyperledger
Fabric channel.

## Channels

In Hyperledger Fabric, each [channel](../channels.html) has a completely
separate ledger. This means a completely separate blockchain, and completely
separate world states, including namespaces. It is possible for applications and
smart contracts to communicate between channels so that ledger information can
be accessed between them.

You can read more about how ledgers work with channels in this
[topic](./developapps/chaincodenamespace.html#channel).


## More information

See the [Transaction Flow](../txflow.html),
[Read-Write set semantics](../readwrite.html) and
[CouchDB as the StateDatabase](../couchdb_as_state_database.html) topics for a
deeper dive on transaction flow, concurrency control, and the world state
database.

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/ -->
