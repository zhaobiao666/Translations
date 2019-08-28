服务发现
=================

为什么我们需要服务发现？
----------------------------------------------

为了在 Peer 节点上执行链码、向排序节点提交交易以及更新交易的状态，应用程序需要连接到 SDK 暴露的 API。

然而，SDK 为了让应用程序连接到网络中对应的节点需要很多信息。除了 CA 已经通道中的排序节点和 Peer 节点的 TLS 证书，包括他们的 IP 地址和端口号，还有它必须知道响应的背书策略以及哪些 Peer 节点安装了链码（这样应用程序才知道应该将链码提案发送到哪些 Peer 节点）。

在 v1.2 之前，这些信息是静态编码的。然而，这种方式不能动态适应网络的变化（比如增加了安装相应链码的 Peer 节点，或者某些节点临时离线）。静态配置同样也不能让应用程序适应背书策略的变化（比如当一个组织加入到通道的时候，可能会改变背书策略）。

另外，客户端应用程序也没办法知道哪些 Peer 节点更新了账本，哪些没有更新账本。应用程序可能会向没有同步账本的 Peer 节点提交提案，会导致交易在提交过程中验证失败，造成资源浪费。

**发现服务** 让 Peer 动态计算需要的信息并且发送给 SDK，从而改进了这个过程。

Fabric 中的服务发现是如何运作的
----------------------------------------------

应用程序在启动时会知道一组应用程序开发者或者管理员信任的 Peer 节点，以便对发现查询提供可信的响应。客户端应用程序的备选节点最好在同一个组织中。注意，为了让 Peer 知道发现服务，必须定义 ``EXTERNAL_ENDPOINT``。要想知道如何定义，请参考 :doc:`discovery-cli` 文档。

应用程序向发现服务发送一个配置查询，来获得所有的静态信息，如果没有获取到，就需要和网络中的其他节点进行通信。这些信息可以通过查询 Peer 节点的发现服务来更新。

发现服务运行在 Peer 节点上，而不是应用程序上，通过使用 gossip 通信层维护的网络元数据信息来查找在线的 Peer 节点。它也会从 Peer 节点的状态数据库中获取信息，比如相关的背书策略。

有了服务发现，应用程序就不需要在指定要哪些 Peer 来背书了。SDK 可以简单地向发现服务发送一个通道名和链码 ID 来查询需要哪些 Peer 节点。发现服务会计算出包含下边两个对象的描述：

1. **分布**：一个 Peer 节点分组的列表和每组被选择的 Peer 节点的数量。

2. **Peer 节点和组的映射**：从分布中的组到通道中的 Peer 节点。实际上，每个组中的节点都应该是同一个组织的，但是因为服务 API 是通用的并且没有区分组织，所以这里用“组”的概念。

下边是 ``AND(Org1, Org2)`` 策略的一个示例，其中每个组织有两个 Peer 节点。

.. code-block:: JSON

   Layouts: [
        QuantitiesByGroup: {
          “Org1”: 1,
          “Org2”: 1,
        }
   ],
   EndorsersByGroups: {
     “Org1”: [peer0.org1, peer1.org1],
     “Org2”: [peer0.org2, peer1.org2]
   }

换句话说，背书策略需要分别来自 Org1 和 Org2 中的一个 Peer 节点的签名。它还提供了这些组织中可用背书节点的名字（Org1 和 Org2 中的 ``peer0`` 和 ``peer1``）。

The SDK then selects a random layout from the list. In the example above, the
endorsement policy is Org1 ``AND`` Org2. If instead it was an ``OR`` policy, the SDK
would randomly select either Org1 or Org2, since a signature from a peer from either
Org would satisfy the policy.

After the SDK has selected a layout, it selects from the peers in the layout based on a
criteria specified on the client side (the SDK can do this because it has access to
metadata like ledger height). For example, it can prefer peers with higher ledger heights
over others -- or to exclude peers that the application has discovered to be offline
-- according to the number of peers from each group in the layout. If no single
peer is preferable based on the criteria, the SDK will randomly select from the peers
that best meet the criteria.

Capabilities of the discovery service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The discovery service can respond to the following queries:

* **Configuration query**: Returns the ``MSPConfig`` of all organizations in the channel
  along with the orderer endpoints of the channel.
* **Peer membership query**: Returns the peers that have joined the channel.
* **Endorsement query**: Returns an endorsement descriptor for given chaincode(s) in
  a channel.
* **Local peer membership query**: Returns the local membership information of the
  peer that responds to the query. By default the client needs to be an administrator
  for the peer to respond to this query.

Special requirements
~~~~~~~~~~~~~~~~~~~~~~
When the peer is running with TLS enabled the client must provide a TLS certificate when connecting
to the peer. If the peer isn't configured to verify client certificates (clientAuthRequired is false), this TLS certificate
can be self-signed.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
