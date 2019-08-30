Frequently Asked Questions
==========================

背书
-----------

**背书架构**:

:Question:
  在一个网络中需要多少个 peers 来对一笔交易进行背书？

:Answer:
  为一笔交易进行背书所需 peers 的数量取决于在 chaincode 部署时期所指定的背书策略。


:Question:
  一个应用程序客户端需要连接所有的 peers 吗？

:Answer:
  客户端只需要连接对于 chaincode 的背书策略所需数量的 peers 即可。

安全 & 访问控制
-------------------------

:Question:
  我如何来确保数据隐私？s

:Answer:
  对于数据隐私，这里有很多个方面。首先，你可以将你的网络隔离到不同的 channels，每个 channel 代表了网络参与者们的一个子集，这些参与者们被授权查看对于被部署到这个 channel 的 chaincode 的数据。

  其次，你可以使用 `private-data <private-data/private-data.html>`_ 来同这个 channel 上的其他组织保持账本的数据隐私。一个 private data collection 允许在一个 channel 上定义的组织的子集有能力来背书、提交或者查询 private data 而不需要创建一个额外的 channel。在这个 channel 上的其他的参与者只会收到数据的哈希值。更多信息可以参考 :doc:`private_data_tutorial` 教程。注意，关键概念话题也解释了 `什么时候应该使用 private data 而不是一个 channel <private-data/private-data.html#when-to-use-a-collection-within-a-channel-vs-a-separate-channel>`_。

  第三，作为 Fabric 使用 private data 来对于数据进行哈希的替代方案，客户端应用能够在调用 chaincode 之前将数据进行哈希或者加密。如果你将数据进行哈希运算的话，那么你需要提供一个方式来共享原始的数据。如果你将信息进行加密的话，那么你需要提供一个方式来共享解密秘钥。


  第四，你可以通过向 chaincode 逻辑中构建访问控制的方式，将对于数据的访问限制在你的组织中的某些角色上。

  第五，剩下的账本数据可以通过在 peer 上的文件系统加密的方式来进行加密，并且在交换的数据是通过 TLS 来进行加密的。

:Question:
   排序节点能够看到交易数据吗？

:Answer:
  不，排序节点仅仅对交易进行排序，他们不会打开交易。如果你不想让数据经过排序节点，那么应该使用 Fabric 的 private data 功能。或者，你可以在调用 chaincode 之前在客户端应用中对数据进行哈希运算或加密。如果你对数据进行加密的话，那么你需要提供 一种方式来共享解密的秘钥。


应用程序端编程模型
----------------------------------

:Question:
  应用程序客户端如何知道一笔交易的输出是什么？

:Answer:
  模拟交易的结果会在提案的反馈中通过背书节点返回给客户端。如果这里有多个背书节点，那么客户端会检查所有的反馈都是一样的，并且将结果和背书提交来进行排序及提交。最终提交节点将会验证或者不验证这笔交易，并且客户端会通过一个 SDK 对于应用程序客户端可用的 event 变得可以了解输出的内容了。

:Question:
  我应该如何查询账本的数据？

:Answer:
  在 chaincode 中，你可以基于 keys 来查询。Keys 可以按照范围来查询，组合 keys 还可以被用来针对于多参数进行等效的查询。比如一个组合 key （owner, asset_id）可以被用来查询所有由某个 entity 所有的资产。这些基于 key 的查询可以被用来对于账本进行只读查询，在更新账本的交易中也可以。

  如果你将资产的数据在 chaincode 中定义为 JSON 格式并且使用 CouchDB 作为 state 数据库的话，你也可以在 chaincode 中使用 CouchDB JSON 查询语言来针对 chaincode 数据值进行复杂的富查询。应用程序客户端可以进行只读查询，但是这些反馈通常不会作为交易的一部分被提交到排序服务。


:Question:
  我应该如何查询历史数据来理解数据的起源？

:Answer:
  Chaincode API ``GetHistoryForKey()`` 能够返回一个 key 对应的值的历史。

:Question:
  如何保证查询的结果是正确的，尤其是当被查询的 peer 可能正在恢复并且在追赶区块的处理？

:Answer:
  客户端可以查询多个 peers，比较他们的区块高度，比较他们的查询结果，并且选择具有更高的区块高度的 peers。

Chaincode（智能合约和数字资产）
----------------------------------------------

:Question:
  Hyperledger Fabric 支持智能合约逻辑吗？

:Answer:
  是的。我们将这个功能称为 Chaincode。它是我们对于智能合约方法/算法的解释，带有额外的功能。

  一个 chaincode 是部署在网络上的编程代码，它会在共识过程中被 chain 验证者执行并验证。开发者可以使用 chaincodes 来开发业务合约，资产定义，以及共同管理的去中心化的应用。

:Question:
  我如何能够创建一个业务合约？

:Answer:
  通常有两种方式开发业务合约：第一种方式是将单独的合约编码到独立的 chaincode 实例中。第二种方式，也可能是更有效率的一种方式，是使用 chaincode 来创建去中心化的应用，来管理一个或者多个类型的业务合约的生命周期，并且让用户在这些应用中实例化这些合约的实例。

:Question:
  我应该如何创建资产？

:Answer:
  用户可以使用 chaincode（对于业务规则）和成员服务（对于数字 tokens）来设计资产，以及管理这些资产的逻辑。

  在大多区块链解决方案中由两种流行的方式来定义资产：stateless 的 UTXO 模型，账户余额会被编码到过去的交易记录中；和账户模型，账户的余额会被保存在账本上的 state 存储空间中。

  每种方式都带有他们自己的好处及坏处。这个区块链技术不主张任何一种方式。相反，我们的第一个需求就是确保两种方式都能够被轻松实现。

:Question:
  编写 chaincode 都支持哪些语言？
  
:Answer:
  Chaincode 能够使用任何的编程语言来编写并且在容器中执行。当前，支持 Golang、node.js 和 java chaincode。

  也可以使用 `Hyperledger Composer <https://hyperledger.github.io/composer/>`__ 来构建 Hyperledger Fabric 应用。

:Question:
  Hyperledger 有原生的货币吗？

:Answer:
  没有。然而，如果你的 chain 网络真的需要一个原生的货币的话，你可以通过 chaincode 来开发你自己的原生货币。对于原生货币的一个常用属性就是一些数量的货币会在每次一笔交易在它的 chain 上被处理的时候被交换（定义该货币的 chaincode 将被调用）。

近期 Releases 中的不同
-----------------------------------

:Question:
  我在哪里能够看到在不同的 releases 中都有哪些变动？

:Answer:
  在任何 releases 中的变动的地方在 :doc:`releases` 中一起被提供出来。

:Question:
  如果上边的问题没有被回答的话，我应该到哪里来获得技术上的帮助？

:Answer:
  请使用 `StackOverflow <https://stackoverflow.com/questions/tagged/hyperledger>`__。


排序服务
----------------

:Question:
  **  我有一个正在运行的排序服务，如果我想要转换共识算法，我该怎么做？**

:Answer:
  这个是不支持的。

..

:Question:
  **什么是排序节点系统通道?**

:Answer:
  排序节点系统通道（有时被称为排序系统通道）指的是排序节点初始被启动的通道。它被用来编排通道的创建。排序节点系统通道定义了联盟以及对于新的通道的初始配置信息。在通道被创建的时候，对于在联盟中的组织的定义， ``/Channel`` 组的值和策略，以及 ``/Channel/Orderer`` 组的值和策略，会被合并起来来形成一个新的初始的通道定义。

..

:Question:
  **如果我更新了我的应用程序 channel，我是否需要更新我的排序系统 channel？**

:Answer:
  一旦一个应用程序 channel 被创建，它会独立于其他任何的 channel（包括排序节点系统 channel）被管理。基于所做的改动，变动可能需要或者可能不需要被放置到其他的 channels。大体来讲，MSP 的变动应该被同步到所有的 channels，然而策略的变动更像是针对于一个特定 channel 的。

..

:Question:
  **我可以有一个组织既作为一个排序节点又作为应用程序的角色吗？**

:Answer:
  尽管这是可能的，但是我们强烈不建议这样配置。默认的 ``/Channel/Orderer/BlockValidation`` 策略允许任何具有有效的证书的排序组织可以来为区块签名。如果一个组织既作为一个排序节点又是应用程序的角色的话，那么这个策略就应该被更新为将区块签名者限制为被授权来排序的证书的子集。

..

:Question:
  **我想要写一个针对于 Fabric 的共识实现，我应该如何开始？**

:Answer:
  一个共识的插件需要实现在 `consensus package`_ 中定义 ``Consenter`` 和 ``Chain`` 接口。针对于这些接口已经有了两个插件：solo_ 和 kafka_。你可以学习他们来为你自己的实现寻求线索。排序服务的代码可以在 `orderer package`_ 中找到。

.. _consensus package: https://github.com/hyperledger/fabric/blob/master/orderer/consensus/consensus.go
.. _solo: https://github.com/hyperledger/fabric/tree/master/orderer/consensus/solo
.. _kafka: https://github.com/hyperledger/fabric/tree/master/orderer/consensus/kafka
.. _orderer package: https://github.com/hyperledger/fabric/tree/master/orderer

..

:Question:
  **我想要改变我的排序服务配置，比如批处理的超时时间，当我启动了网络之后，我该如何做？**

:Answer:
  这属于网络的配置。请参考 :doc:`commands/configtxlator` 话题。

Solo
~~~~

:Question:
  **我如何在生产环境部署 Solo？**

:Answer:
  Solo 不是被用于生产环境的。它不是并且永远也不会是容错的。


Kafka
~~~~~

:Question:
  **我如何能够从排序服务中删除一个节点？**

:Answer:
  这是一个两步的流程：

  1. 将节点的证书添加到相关排序节点的 MSP CRL 中来阻止 peers/客户端连接到它。
  2. 通过使用标准的 Kafka 访问控制措施，比如 TLS CRLs 或者防火墙的方式来阻止节点连接 Kafka 集群。

..

:Question:
  **我之前从来没有部署过一个 Kafka/ZK 集群，我想使用基于 Kafka 的排序服务。我应该如何做？**

:Answer:
  Hyperledger Fabric 文档假设阅读者大体上已经有了维护的经验来创建、设置和管理一个 Kafka 集群（查看 :ref:`kafka-caveat`）。如果没有这样的经验你还要继续的话，你应该在尝试基于 Kafka 的排序服务之前完成，至少 `Kafka Quickstart guide`_ 中的前 6 步。你也可以查看 `this sample configuration file`_ 来了解一个关于 Kafka/ZooKeeper 的合理默认值的简洁的解释。

.. _Kafka Quickstart guide: https://kafka.apache.org/quickstart
.. _this sample configuration file: https://github.com/hyperledger/fabric/blob/release-1.1/bddtests/dc-orderer-kafka.yml

..

:Question:
  **我从哪里能够找到使用基于 Kafka 的排序服务的 Docker？**

:Answer:
  查看 `the end-to-end CLI example`_.

.. _the end-to-end CLI example: https://github.com/hyperledger/fabric/blob/release-1.3/examples/e2e_cli/docker-compose-e2e.yaml

..

:Question:
  **为什么在基于 Kafka 的排序服务中会有对于 ZooKeeper 的依赖？**

:Answer:
  Kafka 在内部使用它来在它的 brokers 之间进行协调。

..

:Question:
  **我尝试遵循 BYFN 样例并且遇到一个 “service unavailable” 错误，我应该怎么做？**

:Answer:
  Check the ordering service's logs. A "Rejecting deliver request because of
  consenter error" log message is usually indicative of a connection problem
  with the Kafka cluster. Ensure that the Kafka cluster is set up properly, and
  is reachable by the ordering service's nodes.

BFT
~~~

:Question:
  **什么时候会有 BFT 版本的排序服务？**

:Answer:
  目前还没有具体的时间。我们在 1.x 周期中尝试将它放置到一个 release 中，比如它会在 Fabric 的一个小的版本更新中。可以查看 FAB-33_ 来获得更新。

.. _FAB-33: https://jira.hyperledger.org/browse/FAB-33

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
