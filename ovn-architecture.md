# OVN 架构

[源文档连接](https://github.com/ovn-org/ovn/blob/master/ovn-architecture.7.xml)

## 概述

OVN(Open Virtual Network) 是一个为虚拟机和容器环境提供逻辑网络抽象的系统。
OVN 在 OVS 已有的能力之上增加了对逻辑网络抽象的原生支持，例如：L2/L3 的 overlay 网络，
安全组（Security Group）以及 DHCP 等网络服务。和 OVS 的设计目标相同，OVN 也是针对生产环境而实现，
并且能在超大规模下顺利运转。

物理网络包括物理的线缆，交换机以及路由器。虚拟网络将物理网络扩展到虚拟机或容器平台，将虚拟机或容器
桥接到物理网络中。OVN 中的虚拟网络通过隧道或者封装的方式以软件来实现，这样这个网络可以和物理网络
或者其他虚拟网络相互隔离。这样逻辑网络中的地址空间可以和物理网络相互重叠并且不会造成冲突。逻辑网络的拓扑
可以不依赖底层物理网络的拓扑。因此逻辑网络中的虚拟机可以从一台物理机迁移到另一台物理机而不会造成网络中断。
请参考下面“逻辑网络”部分获取更多信息。

网络的封装层可以阻止逻辑网络中的虚拟机或容器直接访问底层物理网络的节点。对于虚拟机或者容器机器，通常
这是一个可接受甚至预期的特性。但是有些情况下虚拟机和容器需要能直接访问物理网络。OVN 提供了多种形式的
网关（Gateway）来完成这个目的。请参考下面“网关”部分获取更多信息。


OVN 的主要组成部分

- 云管系统(CMS, Cloud Management System),它是 OVN 的最终客户（用户或管理员通过 CMS 来使用 OVN）。
CMS 如果要集成 OVN，需要和 CMS 相关的插件以及相关软件。OVN 最早的目标 CMS 是 OpenStack。通常我们将 CMS 和 OVN 一一对应，但是在一些场景下，多个 CMS 可以共享使用同一个 OVN。

- 部署 OVN 数据库的物理或虚拟节点，在集群数据库模式下可能是多个节点。

- 一或多个 hypervisor (承载虚拟机或者容器的节点)。Hypervisor 上必须运行 OVS 以及其他相关软件

- 零或多个网关。网关通过在物理网络的以太网端口和隧道端口之间双向转发数据包来打通逻辑网络和物理网络。
这也使得非虚拟化的机器可以加入逻辑网络。网关可以是物理机，虚拟机也可以是支持 vtep 协议的 ASIC 硬件交换机。

Hypervisor 和网关被合称为传输节点（Transport node）或者 chassis。

下图展示了OVN主要组件和相关软件的交互过程。从上到下分别是：

- 云管系统
- CMS 和 OVN 进行交互的 OVN/CMS 插件，在 OpenStack 中这个插件是 Neutron。插件的主要作用是将
存储在 CMS 数据库中， CMS 相关的逻辑网络的配置翻译为 OVN 可以理解的语言。由于每个 CMS 关于逻辑网络
的配置不同，需要针对每个CMS专门开发相应的网络插件。CMS 插件之下的组件都是和 CMS 无关的。

- OVN 北向数据库（Northbound Database）,接受从 CMS 插件下发的逻辑网络配置。北向数据库的数据模式
应该和 CMS 中的概念是“阻抗匹配（impedance matched, 可以理解为一一对应）”的，因此在北向数据库中直接
支持逻辑交换机，逻辑路由器，访问规则控制等概念。请参考 ovn-nb 的文档获取更多信息。OVN 北向数据库只有两个客户，在其上游的 CMS 插件，以及下游的 ovn-northd。

- ovn-northd 连接其上游的 OVN 北向数据库和其下游的 OVN 南向数据库。它将北向数据库中常规意义的网络配置
翻译成南向数据库中的逻辑数据链路（logical datapath）。

OVN 南向数据库的性能必须能够适应不断增加的传输节点。我们现在发现了 ovsdb-server 存在性能瓶颈需要去攻克。
此外 ovsdv-server 还需要有集群模式的高可用能力。

下面几个组件需要在每台 hypervisor 上运行：

- ovn-controller: 作为 OVN 的 agent 和软件网关运行在每个 hypervisor 上。在北向，ovn-controller连接 OVN 南向数据库，
来学习 OVN 的配置和状态变化，并将 hypervisor 的状态更新到 PN 表以及 Binding 表的 chassis 列。在南向，ovn-controller 
作为一个 OpenFlow 控制器连接 ovs-vswitchd 来控制网络流量，并通过本地的 ovsdb-server 来监控和管理 Open vSwitch 的配置不同，需要针对每个CMS专门开发相应的网络插件。CMS

- ovs-vswitchd 和 ovsdb-server 是常规的 Open vSwitch 组件

```bash
                                  CMS
                                   |
                                   |
                       +-----------|-----------+
                       |           |           |
                       |     OVN/CMS Plugin    |
                       |           |           |
                       |           |           |
                       |   OVN Northbound DB   |
                       |           |           |
                       |           |           |
                       |       ovn-northd      |
                       |           |           |
                       +-----------|-----------+
                                   |
                                   |
                         +-------------------+
                         | OVN Southbound DB |
                         +-------------------+
                                   |
                                   |
                +------------------+------------------+
                |                  |                  |
  HV 1          |                  |    HV n          |
+---------------|---------------+  .  +---------------|---------------+
|               |               |  .  |               |               |
|        ovn-controller         |  .  |        ovn-controller         |
|         |          |          |  .  |         |          |          |
|         |          |          |     |         |          |          |
|  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
|                               |     |                               |
+-------------------------------+     +-------------------------------+
```

### OVN 中的信息流向

在 OVN 中配置信息从北向南流动。CMS 通过 OVN/CMS 插件将逻辑网络配置下发给北向数据库。接下来 ovn-northd 
将北向数据库的信息编译成低级语言形式的逻辑流表配置下发到南向数据库，南向数据库再将逻辑流表下发至每个 chassis。

在 OVN 中状态信息反向从南向北流动。OVN 目前只提供少量的状态信息。首先是逻辑交换机端口的 up 状态，ovn-northd
将会更新这个字段。如果南向数据库中这个逻辑接口在 Port_Binding 表的 chassis 字段非空，则 ovn-northd 会将
北向数据库中的 up 字段设置为 true，否则为 false。通过这种方式，CMS 可以探测虚拟机的网络是否就绪。

其次，OVN 提供了让 CMS 了解它的配置是否生效的反馈机制。这需要 CMS 参与序列号协议之中，该协议的工作方式如下：

1. 当 CMS 更新北向数据库配置时，在同一个事务中需要将 NB_Global 表中的 nb_cfg 值加一。（只在CMS希望知道配置
是否最终下发成功时需要这样做）

2. 当 ovn-northd 将某个北向数据库的快照更新至南向数据库时，ovn-northd 会在同一个事务中将北向数据库中 NB_Global 表的 nb_cfg
值更新至南向数据库的 SB_Global 表。（因此如果一个观察者同时监控两个数据库就可以确认南向数据库是否已经和北向数据库同步）

3. 当 ovn-northd 收到南向数据库事务已提交的确认信息后，它会将北向数据库 NB_Global 表的 sb_cfg 字段更新为所下发
到南向数据库快照中的 nb_cfg 的值。（因此 CMS 或者观察者可以无需监控南向数据库，只监控北向数据库的这个值就可以确定数据库是否已经同步）

4. 在每个 chassis 上的 ovn-controller 接收到了南向数据库的更新，同时得到了最新的 nb_cfg 的值。接下来
ovn-controller 更新 chassis 上 Open vSwitch 实例的流表。当收到流表更新成功的确认信息后，ovn-controller 将会
更新南向数据库中对应 Chassis 记录的 nb_cfg 字段。

5. ovn-northd 监控所有 Chassis 记录的 nb_cfg 字段。它将所有 nb_cfg 字段中的最小值复制到北向数据库 NB_Global
表的 hv_cfg 字段。（这样一来，CMS 或者其他观察者只需要监控这个字段就可以确认是否所有 hypervisor 是否已经更新至最新的配置）

### Chassis 设置

在 OVN 中，每个 Chassis 上的 Open vSwitch 都必须有一个专门的网桥（integration bridge）供 OVN 使用。
系统启动脚本会在 ovn-controller 启动时创建这个网桥，如果这个网桥不存在将会按照下列的配置自动创建，默认为 br-int 网桥。在 integration bridge
上有一下几种类型的端口：

- 隧道端口，OVN 维护这些端口来保证逻辑网络的连通性。由 ovn-controller 来负责增加，删除以及更改这些隧道端口。

- 虚拟接口（VIF）,在每个 hypervisor 上接入逻辑网络。

- Gateway 上的物理接口。系统启动脚本会在 ovn-controller 启动前将这些端口加入到网桥中。在一些更复杂的场景中，
也可能是连接到另一个网桥的 patch 端口。

其他类型的端口不应该被加入这个 integration bridge。接入底层 underlay 的物理端口（gateway port相反，是接入逻辑网络的物理端口）
绝对不应该接入这个网桥。实际上 underlay 的物理端口不应该接入任何一个网桥。

Integration bridge 应该按照如下的配置启动，每个配置的具体作用可以参考 ovs-vswitchd.conf.db(5):

- fail-mode=secure

禁止相互隔离的逻辑网络在 ovn-controller 启动前转发数据包。参考 ovs-vsctl(8)中的 Controller Failure Settings
获取更多信息。

- other-config:disable-in-band=true

禁止在 integration bridge 上生成 in-band flow。

Integration bridge 的默认名为 br-int，用户也可以自定义使用其他名字。

### 逻辑网络

OVN 中的逻辑网络概念包含了逻辑交换机和逻辑路由器，分别对应以太网交换机和 IP 路由器。和物理上的概念类似，逻辑交换机和逻辑路由器
可以相互链接构成复杂的网络拓扑。逻辑交换机和逻辑路由器是纯粹的逻辑实体，并不和特定的物理设备绑定，在实现上他们会分布在每一个参与OVN
网络的 hypervisor 上。

逻辑交换机端口（LSPs, Logical switch ports）是逻辑交换机的接入点。OVN 支持多种类型的逻辑交换机端口。再常见的类型是 VIF，这是
虚拟机和容器使用的接入点。一个 VIF 类型的逻辑端口需要和 VM 或容器所在的物理位置绑定，当 VM 发生迁移时绑定关系也会发生变化。（一个 VIF
类型的逻辑端口可以和挂起货值关机中的 VM 关联。这种类型的逻辑端口不会有绑定的物理位置也无法联通）

逻辑路由器端口（LRPs，Logical router ports）是逻辑路由器的接入点。一个 LRP 可以将一个逻辑路由器和一个交换机
或者另一个逻辑路由器相连。虚拟机，容器和其他网络节点只能通过逻辑交换机间接地接入逻辑路由器。

由于逻辑交换机和逻辑路由器有着完全不同类型的逻辑端口，因此当我们讨论"逻辑端口"时，需要明确究竟是是逻辑交换机端口
还是逻辑路由器端口。通常情况下当提及"逻辑端口"时，指的是逻辑交换机端口。

当虚拟机将数据包发送至 VIF 逻辑交换机端口后，Open vSwitch 的流表将会模拟数据包在逻辑链路中交换机和路由器的行为。
流表将会模拟所有的交换和路由行为，而无需其他物理介质参与。如果流表处理最终的结果是将数据包发送至另外一个 hypervisor（或者
其他类型的传输节点）时，数据包会被进行封装并在物理网络上传输。

#### 逻辑交换机端口类型

OVN 支持多种类型的逻辑交换机端口。如上所述 VIF 端口接入虚拟机和容器，是最为常见的一种 LSP 类型。在 OVN 的北向
数据库中，VIF 类型端口的 type 字段值为一个空的字符串。本节将会介绍其他类型的端口。

`router` 类型的逻辑交换机端口将一个逻辑交换机接入一个逻辑路由器，他对段的端口是一个逻辑路由器端口。

`localnet` 类型的逻辑交换机端口将一个逻辑交换机桥接如一个物理 VLAN。一个逻辑交换机可以有一个或多个 `localnet` 类型的端口。
这种类型的逻辑交换机主要在以下两个场景中使用。

- 通过一个或多个 `router` 类型的逻辑交换机端口将 L3 gateway 和分布式 gateway 接入物理网络

- 通过一个或多个 VIF 类型的逻辑交换机端口，将虚拟机或者路由器直接接入物理网络。在这种情况下，逻辑交换机并不是完全逻辑上的，
事实上它桥接进了物理网络，并没有隔离物理网络和虚拟网络，因此也无法拥有独立的 IP 地址空间。使用这种方式可以利用 OVN 控制平面
的能力例如端口安全和 ACL 来管理物理网络。

如果一个逻辑交换机包含多个 `localnet` 类型端口，需要作如下的配置工作：

- 每个 chassis 的 bridge mapping 配置中只能映射其中一个 `localnet` 物理网络
- 为了保证在不同物理网络中 chassis 上 VIF 端口的连通性，底层的物理网络设备需要保证这些物理网络的 L3 连通性。

*注意*：这并不意味着每个 chassis 只能接入一个物理网络，如果多个逻   辑交换机分别属于不同的物理网络，chassis 也可以接入多个物理网络。

`localport` 类型的逻辑交换机端口实例类特殊的 VIF 端口。 这些端口并不和特定 chassis 绑定，会出现在每个 chassis 上。
目的为这些端口的流量不会通过隧道发送到其他机器，只会被发送到同一个 chassis 对应的端口上。OpenStack Neutron 利用 `localport`
来运行元数据服务。元数据代理程序会运行在每个主机上，在同一个逻辑网络的虚拟机可以使用相同的 IP/MAC 地址进行访问，而流量不会跨越节点。
更多细节请参考 OpenStack 中关于 networking-ovn 的文档。

`vtep` 和 `l2gateway` 类型的逻辑交换机端口主要在网关中使用，我们将会在下面的"网关"一节中详细介绍。

#### 实现细节

用户和管理员可能会对上述概念在 OVN 中的内部实现细节感兴趣。

OVN 在南向数据库中通过 `logical datapath` 来实现逻辑网络的细节。ovn-northd 将北向数据库中的每个逻辑交换机
和路由器翻译成南向数据库中的 Datapath_binding 表中的 logical datapath。

ovn-northd 还会将北向数据库中的逻辑交换机端口翻译成南向数据库 Port_Binding 表中的记录。后者基本上和
北向数据库中 Lobical_switch_Port 表一一对应。包含了 VIF，localnet，localport，vtep 和 l2gateway 几种
类型的端口。

此外，Port_Binding 表中还包含几个 lsp 中不存在的类型端口。常见的有 patch 类型 port binding，也常被
称为 `logical patch port`。这些端口会成对出现，数据包流入其中一个端口后从成对的另一个端口流出。ovn-northd利用这些
logical patch port 连接交换机和路由器。

vtep,l2gateway,l3gateway 和 chassisredirect 类型的 port bindings 在网关中使用。请参考下面"网关"部分内容。

### 网关

网关可以提供物理网络和逻辑网络的连通性也可以联通多个不同的 OVN 集群。本节主要关注物理网络和逻辑网络之间的联通，
多个 OVN 之间的联通请参考"OVN Interconnection"

OVN 支持多类型网关。

#### VTEP 网关

`VTEP 网关` 将一个 OVN 逻辑网络接入一个实现 OVSDB VTEP 协议的物理（或者虚拟）交换机。（VTEP 是一个被误用的单词，他本来
只是 VXLAN Tunnel Endpoint 的简写）。请参考"VTEP 网关生命周期"一节获取更多信息。

VTEP 网关的主要使用场景是将物理服务器通过支持 OVSDB VTEP 协议的物理 TOR 交换机加入 OVN 的逻辑网络。

#### L2 网关

L2 网关可以将某个 chassis 上可用的物理 L2 分段加入到逻辑网络中，这样物理网络可以成为逻辑网络的一部分。

为了创建一个 L2 网关，CMS 需要在相关的逻辑交换机上增加一个 `l2gateway` 类型的 LSP，并将它和对应的 chassis 绑定。
ovn-northd 会将这些配置复制到南向数据库的 Port_Binding 表中。在对应的 chassis 上 ovn-controller 会叫数据部
正确的转发到物理分段上。


L2 网关的一些功能和 localnet 端口类似。但是对于 localnet 端口来说，物理网络成为了hypervisor 之间的传输网络。而
对于 L2 网关，hypervisor 之间的数据包仍然要经过隧道， l2gateway 端口只是用来传输需要讲过物理网络的数据包。因此 L2 网关
类型的应用和 VTEP 网关存在相似之处，将非虚拟化的机器加入了物理网络，不过 L2 网关并不依赖特定的 TOR 交换机硬件支持。

#### L3 网关路由器

如"逻辑网络"中所提到的，普通的 OVN 逻辑路由器是分布式运行在每个 hypervisor chassis 上，而不是和某个单独的 chassis 绑定。
这种模式对于有状态服务，例如 SNAT 和 DNAT，这类需要集中式运行的服务来说是个问题。

为了运行这一类有状态服务，OVN 支持 L3 网关路由器，这一类 OVN 逻辑路由器会运行在特定的 chassis 上。网关路由器通常
用来连接分布式逻辑路由器和物理网络。运行在每个 hyperyisor 上的虚拟机和容器先接入逻辑交换机再接入分布式逻辑路由器。
网关路由器通过另一个逻辑交换机和分布式路由器相连，通常这个逻辑交换机也被称为"join"交换机。（OVN 支持多个逻辑路由器不通过中间交换机直接相连，但是
网关路由器只能和逻辑交换机相连。使用 join 逻辑交换机可以减少分布式路由器所需的 IP 数量。）网关路由器相连的另一端的逻辑交换机
拥有一个 localnet 类型的端口接入物理网络。

下图展示了一个典型的场景。多个逻辑交换机 LS1,...,LSn 和分布式逻辑路由器 LR1 相连，LR1 通过 LSjoin 和网关路由器 GLR 相连，
GLR 在另一端和逻辑交换机 LSlocal 相连，在 LSlocal 上有一个 localnet 端口和物理网络相连。

```bash
                                LSlocal
                                   |
                                  GLR
                                   |
                                LSjoin
                                   |
                                  LR1
                                   |
                              +----+----+
                              |    |    |
                             LS1  ...  LSn
```

为了配置一个 L3 网关路由器，CMS需要在北向数据库 "Logical_Router" 实例中设置 "options：chassis"，将其设置成 chassis 的名字。
接下来 ovn-northd 会自动在南向数据库中设置一个类型为 l3gateway 的特殊 port binding 将逻辑路由器和它的邻居相连。接下来，ovn-controller
会将流向这个逻辑路由器的数据包通过隧道发送到特定的机器而不是像分布式路由器那样本地处理。

DNAT 和 SNAT 规则需要和网关路由器关联，它提供了一个集中的地方来处理一对多的 SNAT（IP masquerading）。下面将要介绍的分布式网关端口同样支持 NAT。

#### 分布式网关端口

分布式网关端口是一类特殊的逻辑路由器端口，它指向一个特定的 chassis 进行集中处理，这个 chassis 也被称为 gateway chassis。
分布式网关端口需要连接一个能够连接外部网络端口的逻辑路由器，这个端口可以是 localnet LSP 也可以是接入另一个 OVN 集群的端口（参考 OVN Interconnection）。
数据包在流经分布式网关端口时会先尽可能本地处理，直到需要时再通过额外的一跳传递到 gateway chassis 处理。

下图展示了分布式网关端口的使用方式。

```bash
                                LSlocal
                                   |
                                  LR1
                                   |
                              +----+----+
                              |    |    |
                             LS1  ...  LSn
```

ovn-northd 会为分布式网关端口在南向数据库的 Port_Binding 表创建两条记录。其中一个是和 LRP 同名的 patch 端口，另一个是名为
cr-<port> 的 chassisredirect 类型端口。chassisredirect 端口有一个特殊的任务：当数据包发送到这个端口时需要通过隧道发送至
gateway chassis，在 gateway chassis 上，数据包会自动发送到 patch port binding。因此当某些特殊任务需要再 gateway chassis
上执行时流表会将数据发送至这个 port binding。chassisredirect port binding 不会在其他条件下使用。

CMS 可以通过三种方式类配置分布式网关端口。请参考 ovn-nb(5) 文档中 Logical_router_port 部分的 Distributed Gateway Ports 来了解具体操作。

分布式网关端口支持高可用模式。当指定多个 chassis 时，OVN一次只会使用其中一个作为 gateway chassis。OVN 使用
BFD 来探测各个网关的连通性，并选择当前在线且优先级最高的网关。

##### 物理 VLAN MTU 问题

再次考虑如下的拓扑结构：

```bash
                                LSlocal
                                   |
                                  LR1
                                   |
                              +----+----+
                              |    |    |
                             LS1  ...  LSn
```

假设每个逻辑交换机 LS1,...,LSn 通过 分布式网关端口 LR1 和 LSlocal 上的 localnet 端口桥接进物理的 VLAN-tagged 网络。
如果一个数据包来自 LSi 并且目标是外部网络，OVN 将会将其通过隧道发送至 gateway chassis。当数据部通过 LR1 的逻辑路由器流水线时，
可能会讲过 NAT 处理，最终到达 LSlocal 的 localnet 端口。如果网络中所有的物理链路使用相同的 MTU，那么经过隧道的数据包将会带来
MTU 相关的问题：从隧道到 gateway chassis 的额外隧道开销导致数据包无法充分利用物理链路的 MTU。

OVN 提供了两个解决方案，即 `reside-on-redirect-chassis` 和 `redirect-type` 两个选项。这两种方案均需要
每个虚拟交换机 LS1,...,LSn 均包含一个 localnet 类型端口 LN1,...,LNn，并在每个 chassis 上进行映射。这样
数据包可以直接通过 localnet 端口转发而无需隧道。两种方案的不同在于是全部还是部分数据包需要以这种方式发送。两者
最显著的区别在于`reside-on-redirect-chassis`选项更容易配置，`redirect-type`对东西流量更有利。

第一个方案是在逻辑路由器端口上设置 `reside-on-redirect-shassis` 选项。在 LS1 到 LR1 的 LRP 上设置这个选项
会禁止 gateway chassis 之外 chassis 上处理 LS1 到 LR1 的数据包。在 gateway chassis 之外的 chassis 上发往
LR1 的数据包将会被转发到 LN1。这样经过 LR1 逻辑流水线的数据包最终会到达 LSlocal 上的 localnet 端口进行处理。数据包
在过程中不会经过隧道，也就避免了 MTU 问题。

由于 gateway chassis 之外的 chassis 不会处理 LS1 到 LR1 的数据包，这个选项导致的结果就是逻辑路由器 LR1 从分布式变为集中式。
因此，除了南北流量外，东西流量也必须经过 gateway chassis。

不要在 分布式网关端口上设置 `reside-on-redirect-chassis`，在上面的拓扑中，这个选项应该配置在连接 LS1,...,LSn 到
LR1 的 LRP 上。

第二个方案是使用分布式网关端口的 `redirect-tye` 选项。在 `bridged` 上设置这个选项可以让那些重定向到 gateway chassis 的数据包
通过 localnet 发送，而不是隧道。这个选项不会改变非重定向到 gateway chassis 的数据包。

如果要使用 `redirect-type` 的方案，CMS 或者管理员需要再每个 chassis 上配置一个唯一的以太网地址给逻辑路由器，并通过 Open vSwitch
database 里的 `ovn-chassis-mac-mappings` 提供给 ovn-controller 使用。这相比 `reside-on-redirect-chassis` 的配置要复杂的多。

`redirect-type` 选项需要配置在分布式网关端口上。
