私有数据
============

.. note:: 这个话题假设你已经理解了在 `私有数据的文档 <private-data/private-data.html>`_ 所描述的概念上的资料。

私有数据集合的定义
----------------------------------

一个集合的定义包含了一个或者多个集合，每个集合具有一个原则定义列出了在集合中的所有组织，还包括用来控制在背书阶段私有数据的传播所使用的属性，另外还有一个可选项来决定数据是否会被删除。


Beginning with the Fabric chaincode lifecycle introduced with the Fabric v2.0
Alpha, the collection definition is part of the chaincode definition. The
collection is approved by channel members, and then deployed when the chaincode
definition is committed to the channel. The collection file needs to be the same
for all channel members. If you are using the peer CLI to approve and commit the
chaincode definition, use the ``--collections-config`` flag to specify the path
to the collection definition file. If you are using the Fabric SDK for Node.js,
visit `How to install and start your chaincode <https://fabric-sdk-node.github.io/master/tutorial-chaincode-lifecycle.html>`_.
To use the `previous lifecycle process <https://hyperledger-fabric.readthedocs.io/en/release-1.4/chaincode4noah.html>`_ to deploy a private data collection,
use the ``--collections-config`` flag when `instantiating your chaincode <https://hyperledger-fabric.readthedocs.io/en/latest/commands/peerchaincode.html#peer-chaincode-instantiate>`_.

集合定义由下边的属性组成：

* ``name``: 集合的名字.

* ``policy``: 私有数据集合分发策略，它定义了哪些组织的 peers 被允许使用 ``Signature`` 策略语法并且将每个成员包含在一个 ``OR`` 签名策略列表中来将描述的集合数据进行持久化的操作，为了支持读/写交易，私有数据的分发策略必须要定义一个比 chaincode 背书策略更大范围的一个组织的集合，因为 peers 必须要拥有这些私有数据才能来对这些交易提案进行背书。比如，在一个具有 10 个组织的 channel 中，其中 5 个组织可能会被包含在一个私有数据集合的分发策略中，但是背书策略可能会调用其中的任何 3 个组织来进行背书。

* ``requiredPeerCount``: 在 peer 为背书签名并将提案的回应返回给客户端前，每个背书 peer 必须要成功地传播私有数据到达的 peers 的最小数量(在被授权的组织当中)。将要求传播作为背书的一个条件会确保即使如果背书 peer(s) 变得不可用的时候，私有数据在网络中还是可用的。当 ``requiredPeerCount`` 是 ``0`` 的时候，这代表分发并不是 **必须** 的，但是如果 ``maxPeerCount`` 比 0 大的话，这里可能会有一些分发。``requiredPeerCount`` 是 ``0`` 通常并不被建议，因为那会造成如果背书 peer(s) 变得不可用的时候，网络中的私有数据会丢失。通常你会想要要求当背书的时候对于私有数据会有一些分发来确保私有数据在网络中的多个 peers 上会有冗余存储。


* ``maxPeerCount``: 为了数据的冗余存储的目的，每个背书 peer 将会尝试将私有数据分发到的其他的 peers (在被授权的组织中) 的最大数量。如果在背书时间和提交时间之间一个背书 peer 变得不可用的时候，在背书时间还没有收到私有数据的作为集合成员的其他 peers，将能够从私有数据已经传播到的 peers 那里拉取回私有数据。如果这个值被设置为 ``0``，私有数据在背书的时候不会被扩散，这会强制在提交的时候，私有数据需要从所有被授权的 peers 上的背书 peers 拉取。


* ``blockToLive``: 代表了数据应该以区块的形式在私有数据库中存在多久。数据会在私有数据库上对于指定数量的区块存在，在那之后它会被删除，会使这个数据从网络中废弃。为了无限期的保持私有数据，也就是从来也不会删除私有数据的话，将 ``blockToLive`` 属性设置为 ``0``。

* ``memberOnlyRead``: ``true`` 值表示 peers 自动会强制只有属于这些集合成员的组织中的客户端才被允许读取私有数据。如果一个非成员组织的客户端试图执行一个 chaincode 方法来读取私有数据的话，对于 chaincode 的调用会结束并产生错误。如果你想在单独的 chaincode 方法中进行更细微的访问控制的话，可以使用 ``false`` 值。

下边是一个集合定义 JSON 文件的例子，包含关于两个集合定义的数组：

.. code:: bash

 [
  {
     "name": "collectionMarbles",
     "policy": "OR('Org1MSP.member', 'Org2MSP.member')",
     "requiredPeerCount": 0,
     "maxPeerCount": 3,
     "blockToLive":1000000,
     "memberOnlyRead": true
  },
  {
     "name": "collectionMarblePrivateDetails",
     "policy": "OR('Org1MSP.member')",
     "requiredPeerCount": 0,
     "maxPeerCount": 3,
     "blockToLive":3,
     "memberOnlyRead": true
  }
 ]

这个例子使用了来自于 BYFN 样例网络中的组织，``Org1`` 和 ``Org2``。在 ``collectionMarbles`` 定义中的策略对于私有数据授权了两个组织。这个是在 chaincode 数据需要与排序服务节点保持私有化的时候的一种典型配置。然而，在 ``collectionMarblePrivateDetails`` 定义中的策略却将访问控制在了在 channel (在这里指的是 ``Org1``) 中的一个组织的子集。在一个真正的情况中，在 channel 中会有好多组织，在每个集合中的两个或者多个组织间会彼此共享数据。

私有数据的确认
--------------------------

由于私有数据不会被包含在提交到排序服务的交易中，因此也就不会被包含在被分发给 channel 中所有 peers 的区块中，背书节点扮演着一个传播私有数据给其他被授权组织的 peers 的重要的橘色。这确保了即使背书 peers 在他们的背书之后变成不可用的时候，私有数据在 channel 的集合中的可用性。为了辅助这个传播，在集合定义中的 ``maxPeerCount`` 和 ``requiredPeerCount`` 属性控制了在背书的时候传播的程度。

如果背书 peer 不能够成功地将私有数据分发到至少 ``requiredPeerCount`` 要求的那样，它将会返回一个错误给客户端。背书 peer 会尝试将私有数据分发到不同组织的 peers，来确保每个被授权的组织具有私有数据的一个副本。因为交易在链码执行期间还没有被提交，背书 peer 和接收 peers 除了他们的区块链外，还在一个本地的 ``transient store`` 中存储了私有数据的一个副本，直到交易被提交。

私有数据是如何被提交的
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当一个被授权的节点在提交的时候，在他们的瞬时的数据存储中没有私有数据的副本的时候 (或者是因为他们不是一个背书 peer，或者是因为他们在背书的时候通过传播没有接收到私有数据)，他们会尝试从其他的被授权 peer 那里拉取私有数据，*持续一个可配置的时间长度* 基于在 peer 配置文件 ``core.yaml`` 中的 peer 属性 ``peer.gossip.pvtData.pullRetryThreshold``。

.. note:: 这个被询问私有数据的 peer 将只有当提出请求的 peer 是像私有数据分散策略定义的集合中的一员的时候才会返回私有数据。

当使用 ``pullRetryThreshold`` 时候需要考虑的问题：

* 如果提出请求的 peer 能够在 ``pullRetryThreshold`` 内取回私有数据的话，它将会把交易提交到自己的账本 (包括私有数据的哈希值)，并且将私有数据存储在它的 state 数据库中，同其他 channel state 数据进行了逻辑上的分离。

* 如果提出请求的 peer 没能在 ``pullRetryThreshold`` 内取回私有数据的话，它将会把交易提交到自己的账本 (包括私有数据的哈希值)，但是不会存储私有数据。

* 如果某个 peer 对于私有数据是有资格拥有的，但是却没有得到的话，那么那个 peer 将无法为将来引用到这个丢失的私有数据的交易进行背书 - 对于一个主键丢失的链码查询将会被发现 (基于在 state 数据库中对主键的哈希值的显示)，chaincode 将会收到一个错误。

因此，将 ``requiredPeerCount`` 和 ``maxPeerCount`` 设置成足够大的值来确保在你的 channel 中的私有数据的可用性是非常重要的。比如，如果在交易提交之前，每个背书 peer 都变为不可用了，``requiredPeerCount`` 和 ``maxPeerCount`` 属性将会确保私有数据在其他的 peers 上是可用的。

.. note:: 为了让集合能够工作，在夸组织间的 gossip 配置正确是非常重要的。阅读我们的文档 :doc:`gossip`,尤其注意 "anchor peers" 这部分。

 从链码中引用集合
--------------------------------------

有一系列的 `shim APIs <https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim>`_ 是可用的，可以他们来设置和取回私有数据。

相同的链码数据操作也可以应用到通道 state 数据和私有数据上，但是对于私有数据的情况，要指定一个结合名字，同时带有在链码 APIs 中的数据，比如 ``PutPrivateData(collection,key,value)`` 和 ``GetPrivateData(collection,key)``。

一个单一的链码可以引用多个集合。

如何在一个链码提案中传递私有数据
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

因为链码提案被存储在区块链上，不要把私有数据包含在链码提案的主要部分也是非常重要的。在链码提案中有一个特殊的被称为 ``transient`` 的字段，它可以用来从客户端将私有数据 (或者链码将用来生成私有数据的数据) 传递给在 peer 上的链码的调用。链码可以通过调用 `GetTransient() API <https://github.com/hyperledger/fabric/blob/8b3cbda97e58d1a4ff664219244ffd1d89d7fba8/core/chaincode/shim/interfaces.go#L315-L321>`_ 来获取 ``transient`` 字段。这个 ``transient`` 字段会从通道交易中被排除。


私有数据的访问控制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

直到 1.3 版本，基于集合成员的私有数据的访问控制仅仅会被强制于 peers。基于链码提案的提交者所在的组织的访问控制需要编码在链码逻辑中。从 v1.4 开始，一个结合配置选项 ``memberOnlyRead`` 能够自动地强制使用基于链码提案提交者的组织的访问控制。关于集合配置定义以及如何设置他们的更多信息，请查看这个话题的 `Private data collection definition`_ 章节。

.. note:: 如果你想要更细的访问控制，你可以将 ``memberOnlyRead`` 设置为 false。然后你可以在链码中应用你自己的访问控制逻辑，比如通过调用 GetCreator() 链码 API 或者使用客户端身份 `chaincode library <https://github.com/hyperledger/fabric/tree/master/core/chaincode/shim/ext/cid>`__ 。

查询私有数据
~~~~~~~~~~~~~~~~~~~~~

私有集合数据能够像常见的通道数据那样使用 shim APIs 来进行查询：

* ``GetPrivateDataByRange(collection, startKey, endKey string)``
* ``GetPrivateDataByPartialCompositeKey(collection, objectType string, keys []string)``

对于 CouchDB 状态数据库，JSON 内容查询可以使用 shim API 被传入：

* ``GetPrivateDataQueryResult(collection, query string)``

限制：

* 客户端调用执行范围或者富 JSON 查询的链码的时候应该知道，根据上边关于私有数据扩散部分的解释，如果他们查询的 peer 有丢失的私有数据的话，他们可能会接收到结果集的一个子集。客户端可以查询多个 peers 并且比较返回的结果，以确定是否一个 peer 可能会丢失掉结果集中的部分数据。

* 对于在单一的一个交易中既执行范围或者富 JSON 查询并且更新数据是不支持的，因为查询结果无法在以下类型的 peers 上进行验证的：不能访问私有数据的 peers 或者对于那些他们可以访问相关的私有数据但是私有数据是丢失的。如果一个链码的调用既查询又更新私有数据的话，这个提案请求将会返回一个错误。如果你的应用程序能够容忍在链码执行和验证/提交阶段结果集的变动，那么你可以调用一个链码方法来执行这个查询，然后在调用第二个链码方法来执行变更。注意，调用 GetPrivateData() 来获取单独的键值可以跟 PutPrivateData() 调用放在同一个交易中，因为所有的 peers 都能够基于被哈希过的键的版本来验证键的读取。

使用集合索引
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note:: The Fabric chaincode lifecycle being introduced in the Fabric v2.0
         Alpha does not support using couchDB indexes with your chaincode. To use
         the previous lifecycle model to deploy couchDB indexes with private data
         collections, visit the v1.4 version of the `Private Data Architecture Guide <https://hyperledger-fabric.readthedocs.io/en/release-1.4/private-data-arch.html>`_.

:doc:`couchdb_as_state_database` 章节描述了索引能够被应用到 channel 的 state 数据库来启用 JSON 内容查询，在 chaincode 安装阶段，通过将所以打包在一个 ``META-INF/statedb/couchdb/indexes`` 的路径下。类似的，索引页可以被应用到私有数据集合中，通过将所以打包在一个 ``META-INF/statedb/couchdb/collections/<collection_name>/indexes`` 路径下。一个索引的实例可以查看 `这里 <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02_private/go/META-INF/statedb/couchdb/collections/collectionMarbles/indexes/indexOwner.json>`_。

使用私有数据时的思考
--------------------------------------

私有数据删除
~~~~~~~~~~~~~~~~~~~~

Private data can be periodically purged from peers. For more details,
see the ``blockToLive`` collection definition property above.

Additionally, recall that prior to commit, peers store private data in a local
transient data store. This data automatically gets purged when the transaction
commits.  But if a transaction was never submitted to the channel and
therefore never committed, the private data would remain in each peer’s
transient store.  This data is purged from the transient store after a
configurable number blocks by using the peer’s
``peer.gossip.pvtData.transientstoreMaxBlockRetention`` property in the peer
``core.yaml`` file.

Updating a collection definition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To update a collection definition or add a new collection, you can upgrade
the chaincode to a new version and pass the new collection configuration
in the chaincode upgrade transaction, for example using the ``--collections-config``
flag if using the CLI. If a collection configuration is specified during the
chaincode upgrade, a definition for each of the existing collections must be
included.

When upgrading a chaincode, you can add new private data collections,
and update existing private data collections, for example to add new
members to an existing collection or change one of the collection definition
properties. Note that you cannot update the collection name or the
blockToLive property, since a consistent blockToLive is required
regardless of a peer's block height.

Collection updates becomes effective when a peer commits the block that
contains the chaincode upgrade transaction. Note that collections cannot be
deleted, as there may be prior private data hashes on the channel’s blockchain
that cannot be removed.

Private data reconciliation
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Starting in v1.4, peers of organizations that are added to an existing collection
will automatically fetch private data that was committed to the collection before
they joined the collection.

This private data "reconciliation" also applies to peers that
were entitled to receive private data but did not yet receive it --- because of
a network failure, for example --- by keeping track of private data that was "missing"
at the time of block commit.

Private data reconciliation occurs periodically based on the
``peer.gossip.pvtData.reconciliationEnabled`` and ``peer.gossip.pvtData.reconcileSleepInterval``
properties in core.yaml. The peer will periodically attempt to fetch the private
data from other collection member peers that are expected to have it.

Note that this private data reconciliation feature only works on peers running
v1.4 or later of Fabric.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
