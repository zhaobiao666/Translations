Gossip 数据传播协议
==================================

Hyperledger Fabric 通过将工作负载拆分为交易执行（背书和提交）节点和交易排序节点的方式来优化区块链网络的性能、安全性和可扩展性。对网络操作这样的分割就需要一个安全、可靠和可扩展的数据传播协议来保证数据的完整性和一致性。为了满足这个需求，Fabric 实现了 **Gossip 数据传播协议** 。

Gossip 协议
---------------

Peer 节点通过 gossip 协议来广播来传播账本和通道数据。Gossip 消息是持续的，通道中的每一个 Peer 节点不断地从多个节点接受当前一致的账本数据。每一个 gossip 消息都是带有签名的，因此拜占庭成员发送的伪造消息很容易就会被识别，并且非目标节点也不会接受与其无关的消息。Peer 节点会受到延迟、网络分区或者其他原因影响而丢失区块，这时节点会通过从其他拥有这些丢失区块的节点处同步账本。

基于 gossip 的数据传播协议在 Fabric 网络中有三个主要功能：

1. 通过持续的识别可用成员节点来管理节点发现和通道成员，还有检测离线节点。
2. 向通道中的所有节点传播账本数据。所有没有和当前通道的数据同步的节点会识别丢失的区块，并将正确的数据复制过来以使自己同步。
3. 通过点对点的数据传输方式，使新节点以最快速度连接到网络中并同步账本数据。

Peer 节点基于 Gossip 的数据广播操作接收通道中其他的节点的信息，然后将这些信息随机发送给通道上的一些其他节点，随机发送的节点数量是一个可配置的常量。Peer 节点可以用“拉”的方式获取信息而不用一致等待。这是一个重复的过程，以使通道中的成员、账本和状态信息同步并保持最新。在分发新区块的时候，通道中**主**节点从排序服务拉取数据然后开始在它所在的组织的节点中分发。

领导者选举
---------------

领导者的选举机制用于在每一个组织中**选举**出一个用于链接排序服务和初始分发新区块的节点。领导者选举使得系统可以有效地利用排序服务的带宽。领导者选举模型有两种模式可供选择：

1. **静态模式**：系统管理员手动配置一个节点为组织的主节点。
2. **动态模式**：组织中的节点自己选举出一个主节点。

静态主节点选举
~~~~~~~~~~~~~~~~~~~~~~

静态主节点选举允许你手动设置组织中的一个或多个节点节点为主节点。请注意，太多的节点连接到排序服务可能会影响带宽使用效率。要开启静态主节点选举模式，需要配置 ``core.yaml`` 中的如下部分：

::

    peer:
        # Gossip related configuration
        gossip:
            useLeaderElection: false
            orgLeader: true

另外，这些配置的参数可以通过环境变量覆盖：

::

    export CORE_PEER_GOSSIP_USELEADERELECTION=false
    export CORE_PEER_GOSSIP_ORGLEADER=true


.. note:: 下边的设置会使节点进入**旁观者**模式，也就是说，它不会试图成为一个主节点：

::

    export CORE_PEER_GOSSIP_USELEADERELECTION=false
    export CORE_PEER_GOSSIP_ORGLEADER=false

不要将 ``CORE_PEER_GOSSIP_USELEADERELECTION`` 和 ``CORE_PEER_GOSSIP_ORGLEADER`` 都设置为 ``true``，这将会导致错误。

在静态配置组织中，主节点失效或者崩溃都需要管理员进行处理。

动态主节点选举
~~~~~~~~~~~~~~~~~~~~~~~

动态主节点选举使组织中的节点可以**选举**一个节点来连接排序服务并拉取新区块。这个主节点由每个组织单独选举。

动态选举出的的主节点通过向其他节点发送**心跳**信息来证明自己处于存活状态。如果一个或者更多的节点在一个段时间内没有收到**心跳**信息，它们就会选举出一个新的主节点。

在网络比较差有多个网络分区存在的情况下，组织中会存在多个主节点以保证组织中节点的正常工作。在网络恢复正常之后，其中一个主节点会放弃领导权。在一个没有网络分区的稳定状态下，会只有**唯一**一个活动的主节点和排序服务相连。

下边的配置控制主节点**心跳**信息的发送频率：

::

    peer:
        # Gossip related configuration
        gossip:
            election:
                leaderAliveThreshold: 10s

为了开启动态节点选举，需要配置 ``core.yaml`` 中的以下参数：

::

    peer:
        # Gossip related configuration
        gossip:
            useLeaderElection: true
            orgLeader: false

同样，这些配置的参数可以通过环境变量覆盖：

::

    export CORE_PEER_GOSSIP_USELEADERELECTION=true
    export CORE_PEER_GOSSIP_ORGLEADER=false

锚节点
------------

gossip 利用锚节点来保证不同组织间的互相通信。

When a configuration block that contains an update to the anchor peers is committed,
peers reach out to the anchor peers and learn from them about all of the peers known
to the anchor peer(s). Once at least one peer from each organization has contacted an
anchor peer, the anchor peer learns about every peer in the channel. Since gossip
communication is constant, and because peers always ask to be told about the existence
of any peer they don't know about, a common view of membership can be established for
a channel.

For example, let's assume we have three organizations---`A`, `B`, `C`--- in the channel
and a single anchor peer---`peer0.orgC`--- defined for organization `C`. When `peer1.orgA`
(from organization `A`) contacts `peer0.orgC`, it will tell it about `peer0.orgA`. And
when at a later time `peer1.orgB` contacts `peer0.orgC`, the latter would tell the
former about `peer0.orgA`. From that point forward, organizations `A` and `B` would
start exchanging membership information directly without any assistance from
`peer0.orgC`.

As communication across organizations depends on gossip in order to work, there must
be at least one anchor peer defined in the channel configuration. It is strongly
recommended that every organization provides its own set of anchor peers for high
availability and redundancy. Note that the anchor peer does not need to be the
same peer as the leader peer.

External and internal endpoints
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order for gossip to work effectively, peers need to be able to obtain the
endpoint information of peers in their own organization as well as from peers in
other organizations.

When a peer is bootstrapped it will use ``peer.gossip.bootstrap`` in its
``core.yaml`` to advertise itself and exchange membership information, building
a view of all available peers within its own organization.

The ``peer.gossip.bootstrap`` property in the ``core.yaml`` of the peer is
used to bootstrap gossip **within an organization**. If you are using gossip, you
will typically configure all the peers in your organization to point to an initial set of
bootstrap peers (you can specify a space-separated list of peers). The internal
endpoint is usually auto-computed by the peer itself or just passed explicitly
via ``core.peer.address`` in ``core.yaml``. If you need to overwrite this value,
you can export ``CORE_PEER_GOSSIP_ENDPOINT`` as an environment variable.

Bootstrap information is similarly required to establish communication **across
organizations**. The initial cross-organization bootstrap information is provided
via the "anchor peers" setting described above. If you want to make other peers
in your organization known to other organizations, you need to set the
``peer.gossip.externalendpoint`` in the ``core.yaml`` of your peer.
If this is not set, the endpoint information of the peer will not be broadcast
to peers in other organizations.

To set these properties, issue:

::

    export CORE_PEER_GOSSIP_BOOTSTRAP=<a list of peer endpoints within the peer's org>
    export CORE_PEER_GOSSIP_EXTERNALENDPOINT=<the peer endpoint, as known outside the org>

Gossip messaging
----------------

Online peers indicate their availability by continually broadcasting "alive"
messages, with each containing the **public key infrastructure (PKI)** ID and the
signature of the sender over the message. Peers maintain channel membership by collecting
these alive messages; if no peer receives an alive message from a specific peer,
this "dead" peer is eventually purged from channel membership. Because "alive"
messages are cryptographically signed, malicious peers can never impersonate
other peers, as they lack a signing key authorized by a root certificate
authority (CA).

In addition to the automatic forwarding of received messages, a state
reconciliation process synchronizes **world state** across peers on each
channel. Each peer continually pulls blocks from other peers on the channel,
in order to repair its own state if discrepancies are identified. Because fixed
connectivity is not required to maintain gossip-based data dissemination, the
process reliably provides data consistency and integrity to the shared ledger,
including tolerance for node crashes.

Because channels are segregated, peers on one channel cannot message or
share information on any other channel. Though any peer can belong
to multiple channels, partitioned messaging prevents blocks from being disseminated
to peers that are not in the channel by applying message routing policies based
on a peers' channel subscriptions.

.. note:: 1. Security of point-to-point messages are handled by the peer TLS layer, and do
          not require signatures. Peers are authenticated by their certificates,
          which are assigned by a CA. Although TLS certs are also used, it is
          the peer certificates that are authenticated in the gossip layer. Ledger blocks
          are signed by the ordering service, and then delivered to the leader peers on a channel.

          2. Authentication is governed by the membership service provider for the
          peer. When the peer connects to the channel for the first time, the
          TLS session binds with the membership identity. This essentially
          authenticates each peer to the connecting peer, with respect to
          membership in the network and channel.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
