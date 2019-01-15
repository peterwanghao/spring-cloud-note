# 第一节 Consul架构

Consul是由HashiCorp基于Go语言开发的支持多数据中心分布式高可用的服务发布和注册服务软件，采用Raft算法保证服务的一致性，且支持健康检查。

## Consul架构
只有一个数据中心的Consul的架构图如下：
![Architecture](https://img-blog.csdnimg.cn/20181212214953687.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BldGVyd2FuZ2hhbw==,size_16,color_FFFFFF,t_70)

我们可以看到，有三个不同的服务器由Consul管理。整个架构通过使用Raft算法工作，这有助于我们从三个不同的服务器中选出一个领导者。然后根据诸如Follower和Leader之类的标签标记这些服务器。顾名思义，Follower有责任遵循Leader的决定。这三个服务器之间进一步相互连接以进行通信。

每个服务器使用RPC与其自己的客户端进行交互。客户端之间的通信是使用Gossip协议。可以使用TCP或Gossip来提供与互联网设施的通信。

## Raft算法

Raft是提供一致性的算法。它依赖于CAP定理的原理，该定理指出，在存在网络分区的情况下，必须在一致性和可用性之间进行选择。并非CAP定理的所有三个基本原理：Consistency(一致性)、Availability(可用性)、Partition Tolerance(分区容错性)，都可以在任何给定的时间点实现。人们必须在最好的情况下权衡其中任何两个。

一个Raft集群包含多个服务器，通常是奇数的。例如，如果我们有五台服务器，它将允许系统容忍两个故障。在任何给定时间，每个服务器都处于以下三种状态之一：Leader，Follower或Candidate。在正常操作中，只有一个领导者，所有其他服务器都是Follower。这些Follower处于被动状态，即他们自己不发出请求，而只是响应Leader和Candidate的请求。

下图描述了使用Raft算法工作的工作流模型
![Raft算法](https://img-blog.csdnimg.cn/20181212215445859.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BldGVyd2FuZ2hhbw==,size_16,color_FFFFFF,t_70)

## 协议类型

Consul中有两种类型的协议，称为 

 - Consensus协议 
 - Gossip协议

现在让我们详细了解它们。

### Consensus Protocol
Consul使用Consensus protocol来提供CAP定理所描述的一致性。该协议基于Raft算法，其中Raft节点始终处于三种状态中的任何一种：Follower, Candidate or Leader。
- Leader，负责Client交互和log复制，同一时刻系统中最多存在1个
- Follower，被动响应请求RPC，从不主动发起请求RPC
- Candidate，由Follower向Leader转换的中间状态

初始时所有节点都是以Follower启动。一个最小的 Raft 集群需要三个参与者，这样才可能投出多数票。当发起选举时，如果每方都投给了自己，结果没有任何一方获得多数票。之后每个参与方随机休息一阵（Election Timeout）重新发起投票直到一方获得多数票。这里的关键就是随机 timeout，最先从timeout中恢复发起投票的一方向还在 timeout 中的另外两方请求投票，这时它们就只能投给对方了，这样很快就能达成一致。

### Gossip Protocol

Gossip协议可用于管理成员资格，跨群集发送和接收消息。在Consul中，Gossip协议的使用以两种方式发生，WAN（无线区域网络）和LAN（局域网）。有三个已知的库，可以实现Gossip算法来发现对等网络中的节点 ：
  - teknek-gossip - 它与UDP一起使用，用Java编写。
  - gossip-python - 它利用TCP堆栈，也可以通过构建的网络共享数据。
  - Smudge - 它是用Go编写的，使用UDP来交换状态信息。

Gossip协议也被用于实现和维护分布式数据库一致性或与一致状态的其他类型数据，计算未知大小的网络中的节点数量，稳健地传播消息，组织节点等。

## Remote Procedure Calls

RPC可以表示为远程过程调用的简写形式。它是一个程序用于从另一个程序请求服务的协议。此协议可以位于网络上的另一台计算机中，而无需确认网络详细信息。

在Consul中使用RPC的真正用处在于，它可以帮助我们避免大多数发现服务工具在一段时间之前所遇到的延迟问题。在RPC之前，Consul过去只使用基于TCP和UDP的连接，这对大多数系统都很好，但对于分布式系统则不行。RPC通过减少从一个地方到另一个地方的分组信息传输的时间段来解决这些问题。