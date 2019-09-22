能力需求
-----------------------

由于 Fabric 是一个分布式系统，通常会涉及多个组织（有时在不同的国家甚至大洲），所以网络中可能（而且通常）存在许多不同版本的 Fabric。然而，重要的是，网络以相同的方式处理交易，这样每个人对当前网络状态都有相同的看法。

这意味着每个网络，以及该网络中的每个通道，必须定义一组我们称为“能力”（Capabilities）的东西，以便能够参与处理交易。例如，Fabric v1.1引入了新的 MSP 角色类型 “Peer” 和 “Client”。但是，如果1.0版本的节点不理解这些新角色类型，那么它将无法恰当地验证引用它们的背书策略。这意味着在使用新角色类型之前，网络必须同意启用v1.1的 ``通道`` 能力需求，确保所有节点都做出相同的决定。

只有支持所需能力的二进制文件才能参与通道，较新的二进制版本在启用相应能力之前不会启用新的验证逻辑。通过这种方式，能力需求确保即使使用不同的构建和版本，网络也不会出现状态分叉。

定义能力需求
================================

能力需求是在通道配置中为每个通道定义的（可以在通道的最新配置区块中找到）。通道配置包含三个位置，每个位置定义了一个不同类型的能力。

+------------------+-----------------------------------+----------------------------------------------------+
| Capability Type  | Canonical Path                    | JSON Path                                          |
+==================+===================================+====================================================+
| Channel          | /Channel/Capabilities             | .channel_group.values.Capabilities                 |
+------------------+-----------------------------------+----------------------------------------------------+
| Orderer          | /Channel/Orderer/Capabilities     | .channel_group.groups.Orderer.values.Capabilities  |
+------------------+-----------------------------------+----------------------------------------------------+
| Application      | /Channel/Application/Capabilities | .channel_group.groups.Application.values.          |
|                  |                                   | Capabilities                                       |
+------------------+-----------------------------------+----------------------------------------------------+

* **通道（Channel）：** 这些能力适用于 Peer 进度和排序进度，并且位于根 ``Channel`` 组中。
* **排序节点（Orderer）：** 仅适用于排序节点，并且位于 ``Orderer`` 组中。
* **应用程序（Application）：** 仅适用于 Peer 节点，并且位于 ``Application`` 组中。

为了与现有的管理结构保持一致，这些能力被分解到这些组中。更新排序节点的能力是排序组织自己的事，独立于应用组织。类似的，更新应用程序的能力只是应用程序管理员的事情，与排序节点无关。 通过将“排序节点”和“应用程序”之间的能力分离，假设一个网络可以运行在v1.6的排序服务，同时又支持运行在v1.3节点的应用网络。

然而，有些能力跨越了“应用程序”和“排序节点”组。正如我们前面看到的，添加新的 MSP 角色类型是排序节点管理员和应用管理员都同意并且需要认识到的。排序节点必须理解 MSP 角色的含义，以便允许事务通过排序，而 Peer 节点必须理解角色，以便验证事务。这种能力（它跨越了应用和排序节点组件）在顶层“通道”组中定义。

.. note:: 通道能力可能被定义为v1.3版本，而排序节点和应用分别被定义为1.1版本和v1.4版本。在“通道”组级别启用能力并不意味着在特定的“排序节点”和“应用程序”组级别可以使用相同的能力。

设置能力
====================

能力被设置为通道配置的一部分（或者作为初始配置的一部分，或者作为重新配置的一部分）。

.. note:: 我们有两个文档讨论了通道重新配置的不同方面。首先，我们有一个教程 :doc:`channel_update_tutorial` 为您演示  。我们也有另一个文档，讨论了如何 :doc:`config_update`，它给出了各种更新的概述以及对签名过程的更全面的了解。

因为在默认情况下，新通道会复制排序系统通道的配置，因此在新通道创建时会使用和排序系统通道一样的排序节点和通道能力，以及应用程序能力自动配置新通道。然而，已经存在的通道必须重新配置。

能力在 protobuf 中定于的结构如下：

.. code:: bash

  message Capabilities {
        map<string, Capability> capabilities = 1;
  }

  message Capability { }

用 JSON 格式举例：

.. code:: bash

  {
      "capabilities": {
          "V1_1": {}
      }
  }

初始化配置中的能力
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Fabric 源代码中 ``config`` 路径下的 ``configtx.yaml`` 文件中， 在 ``Capabilities`` 部分列举了每种能力类型（Channel、Orderer 和 Application）。

启用能力的最简单方法是选择一个v1.1示例概要文件并为您的网络定制它。例如:

.. code:: bash

    SampleSingleMSPSoloV1_1:
        Capabilities:
            <<: *GlobalCapabilities
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *SampleOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *SampleOrg

注意，在根级别（用于通道能力)和在排序级别（用于排序能力）定义了一个 ``Capabilities`` 部分。 上面的示例使用 YAML 引用的方式将定义在文件底部的 capabilities 部分包含进来。

在定义排序系统通道时，不存在应用程序部分，因为这些能力是在创建应用程序通道时定义的。 要在通道创建时定义新通道的应用程序能力，应用程序管理员应该在 ``SampleSingleMSPChannelV1_1`` 中对其通道创建事务建模。

.. code:: bash

   SampleSingleMSPChannelV1_1:
        Consortium: SampleConsortium
        Application:
            Organizations:
                - *SampleOrg
            Capabilities:
                <<: *ApplicationCapabilities

Applicatoin 部分的 ``Capabilities`` 元素引用了定义在 YAML 文件底部的 ``ApplicationCapabilities`` 部分。

.. note:: The capabilities for the Channel and Orderer sections are inherited from
          the definition in the ordering system channel and are automatically included
          by the orderer during the process of channel creation.

.. note:: 应用通道中的通道和排序节点能力继承自排序系统通道中的定义，在创建通道的时候被自动包含进来。

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
