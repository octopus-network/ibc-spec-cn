---
ics: '18'
title: 中继器算法
stage: 草案
category: IBC/TAO
kind: 接口
requires: 24, 25, 26
author: Christopher Goes <cwgoes@tendermint.com>
created: '2019-03-07'
modified: '2019-08-25'
---

## 概要

中继器算法是 IBC 的“物理”连接层——链下进程通过扫描链的状态，构造适当的数据报，并按照 IBC 规定在对方链上执行，从而在运行 IBC 协议的两条链之间中继数据。

### 动机

在 IBC 协议中，区块链只能记录将特定数据发送到另一条链的*意图*——它不能直接访问网络传输层。物理数据报中继必须由能够访问传输层（例如 TCP/IP）的链下基础设施执行。该标准定义了*中继*算法的概念，该算法可由具有查询链状态的链下进程执行，以执行此中继。

### 定义

*中继器*是一种链下进程，能够使用 IBC 协议读取状态并将交易提交到某些账本集。

### 所需属性

- IBC 的仅一次传递或超时安全属性都不应依赖中继器的行为（假设中继器可以有拜占庭行为）。
- IBC 的中继活性仅应依赖于至少一个正确的，活跃的中继器存在。
- 中继应该是不需许可的，所有必要的验证都应在链上执行。
- 应该最小化 IBC 用户和中继器之间的必要通信。
- 应能在应用层提供中继器激励措施。

## 技术指标

### 基础中继器算法

中继器算法是在一个实现了 IBC 协议的链集`C`上定义的。每个中继器不一定需要访问链间网络中所有链的状态来读取数据报或将数据报写入链间网络中的所有链（尤其是在许可链或私有链的情况下），不同的中继器可以在不同子集之间中继。

`pendingDatagrams`根据两条链的状态计算要从一个链中继到另一个链的所有有效数据报的集合。中继器必须具有为其中继的集合中的区块链实现了哪些 IBC 协议的子集的先验知识（例如，通过阅读源代码）。下面定义了一个示例。

`submitDatagram`是链自己定义的过程（提交某个交易）。数据报可以每个当作单独的交易提交，也可以在链支持的情况下作为一整个交易原子性提交。

`relay`每隔一段时间就会调用一次 - 不高于任一链的出块速度，并且可能根据中继器期望的中继频率而降低一些。

不同的中继器可以在不同的链之间进行中继——只要每对链具有至少一个正确且活跃的中继器，同时这些链保持活性，网络中链之间流动的所有数据包最终都将被中继。

```typescript
function relay(C: Set<Chain>) {
  for (const chain of C)
    for (const counterparty of C)
      if (counterparty !== chain) {
        const datagrams = chain.pendingDatagrams(counterparty)
        for (const localDatagram of datagrams[0])
          chain.submitDatagram(localDatagram)
        for (const counterpartyDatagram of datagrams[1])
          counterparty.submitDatagram(counterpartyDatagram)
      }
}
```

### 数据包、回执、超时

#### 在有序通道中中继数据包

可以基于事件的方式或基于查询的方式中继有序通道中的数据包。对于前者，中继器应监视源链，每当发送数据包发出事件时，使用事件日志中的数据来组成数据包。对于后者，中继器应定期查询源链上的发送序列号，并保持中继的最后一个序列号，两者之间的任何序列号都是需要查询然后中继的数据包。无论哪种情况，中继器进程都应通过检查接收序列号来检查目的链是否尚未接收到这个数据包，然后才进行中继。

#### 在无序通道中中继数据包

可以基于事件的方式中继无序通道中的数据包。中继器应监视源链中每个发送数据包发出的事件，然后使用事件日志中的数据来组成数据包。随后，中继器应通过查询数据包的序列号是否存在对应的回执来检查目的链是否已接收到过该数据包，如果尚未出现，中继器才中继该数据包。

#### 中继回执

回执可以基于事件的方式进行中继。中继器应该监视目标链，每当接收数据包并写入回执时，使用事件日志中的数据组成回执数据包，检查数据包承诺在源链上是否存在（一旦回执被中继，它将被删除），如果是，则将回执中继到源链。

#### 中继超时

超时中继稍微复杂一些，因为当数据包超时时没有特定事件发出，这是简单的情况，由于目标链已经超过超时高度或时间戳，因此无法再中继数据包。中继器进程必须选择跟踪一组数据包（可以通过扫描事件日志来构造），并且一旦目的链的高度或时间戳超过跟踪的数据包的高度或时间戳，就检查数据包承诺是否仍存在于源链（一旦超时被中继，它将被删除），如果是，则将超时中继到源链。

### 待处理的数据报

`pendingDatagrams`整理要从一台机器发送到另一台机器的数据报。此功能的实现将取决于两台机器支持的 IBC 协议子集和源机器的状态布局。特定的中继器可能还希望实现他们自己的过滤器功能，以便仅中继可能被中继的数据报的子集（例如，他们已支付费用以某种链外方式中继的子集）。

下面概述了在两个链之间执行单向中继的示例实现。通过交换`chain`和`counterparty` ，可以更改为执行双向中继。 哪个中继器进程负责哪个数据报是一个灵活的选择——在此示例中，中继器进程中继在`chain`上开始的所有握手（将数据报发送到两个链），中继从`chain`发送的所有数据包到`counterparty` ，并中继所有数据包的回执从`counterparty`发送到`chain` 。

```typescript
function pendingDatagrams(chain: Chain, counterparty: Chain): List<Set<Datagram>> {
  const localDatagrams = []
  const counterpartyDatagrams = []

  // ICS2 : 客户端
  // - 确定轻客户端是否需要更新（本地和对方）
  height = chain.latestHeight()
  client = counterparty.queryClientConsensusState(chain)
  if client.height < height {
    header = chain.latestHeader()
    counterpartyDatagrams.push(ClientUpdate{chain, header})
  }
  counterpartyHeight = counterparty.latestHeight()
  client = chain.queryClientConsensusState(counterparty)
  if client.height < counterpartyHeight {
    header = counterparty.latestHeader()
    localDatagrams.push(ClientUpdate{counterparty, header})
  }

  // ICS3 : 连接
  // - 确定是否正在进行任何连接握手
  connections = chain.getConnectionsUsingClient(counterparty)
  for (const localEnd of connections) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    if (localEnd.state === INIT &&
          (remoteEnd === null || remoteEnd.state === INIT))
      // 握手已在本地开始（完成 1 步），将 `connOpenTry` 中继到远程端
      counterpartyDatagrams.push(ConnOpenTry{
        desiredIdentifier: localEnd.counterpartyConnectionIdentifier,
        counterpartyConnectionIdentifier: localEnd.identifier,
        counterpartyClientIdentifier: localEnd.clientIdentifier,
        counterpartyPrefix: localEnd.commitmentPrefix,
        clientIdentifier: localEnd.counterpartyClientIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofConsensus: localEnd.client.consensusState.proof(),
        proofHeight: height,
        consensusHeight: localEnd.client.height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // 另一端已开始握手（完成 2 步），将 `connOpenAck` 中继到本地端
      localDatagrams.push(ConnOpenAck{
        identifier: localEnd.identifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofConsensus: remoteEnd.client.consensusState.proof(),
        proofHeight: remoteEnd.client.height,
        consensusHeight: remoteEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // 握手已在本地确认（完成 3 步），将 `connOpenConfirm` 中继到远程端
      counterpartyDatagrams.push(ConnOpenConfirm{
        identifier: remoteEnd.identifier,
        proofAck: localEnd.proof(),
        proofHeight: height,
      })
  }

  // ICS4：通道和数据包
  // - 确定是否正在进行任何通道握手
  // - 确定是否需要中继任何数据包、回执或超时
  channels = chain.getChannelsUsingConnections(connections)
  for (const localEnd of channels) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    // 处理正在进行的握手
    if (localEnd.state === INIT &&
          (remoteEnd === null || remoteEnd.state === INIT))
      // 握手已在本地开始（完成 1 步），将 `chanOpenTry` 中继到远程端
      counterpartyDatagrams.push(ChanOpenTry{
        order: localEnd.order,
        connectionHops: localEnd.connectionHops.reverse(),
        portIdentifier: localEnd.counterpartyPortIdentifier,
        channelIdentifier: localEnd.counterpartyChannelIdentifier,
        counterpartyPortIdentifier: localEnd.portIdentifier,
        counterpartyChannelIdentifier: localEnd.channelIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofHeight: height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // 另一端已开始握手（已完成 2 步），将 `chanOpenAck` 中继到本地端
      localDatagrams.push(ChanOpenAck{
        portIdentifier: localEnd.portIdentifier,
        channelIdentifier: localEnd.channelIdentifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofHeight: localEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // 本地握手已确认（完成 3 步），将 `chanOpenConfirm` 中继到远程端
      counterpartyDatagrams.push(ChanOpenConfirm{
        portIdentifier: remoteEnd.portIdentifier,
        channelIdentifier: remoteEnd.channelIdentifier,
        proofAck: localEnd.proof(),
        proofHeight: height
      })

    // 处理数据包
    // 首先，扫描发送数据包的日志并中继所有数据包
    sentPacketLogs = queryByTopic(height, "sendPacket")
    for (const logEntry of sentPacketLogs) {
      // 用这个序列号中继数据包
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketRecv{
        packet: packetData,
        proof: packet.proof(),
        proofHeight: height,
      })
    }

    // 然后，扫描日志以获取回执，中继回发送链
    recvPacketLogs = queryByTopic(height, "writeAcknowledgement")
    for (const logEntry of recvPacketLogs) {
      // 使用此序列号中继数据包回执
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketAcknowledgement{
        packet: packetData,
        acknowledgement: logEntry.acknowledgement,
        proof: packet.proof(),
        proofHeight: height,
      })
    }
  }

  return [localDatagrams, counterpartyDatagrams]
}
```

中继器可以选择过滤这些数据报，或许会根据费用支付模型，来中继特定的客户端、特定的连接、特定的通道，甚至特定类型的数据包（本文档未指定，因为它可能会有所不同）。

### 排序约束

在中继器进程上存在隐式排序约束，以确定必须以什么顺序提交哪些数据报。例如，必须先提交区块头才能最终确定存储在轻客户端中特定高度的共识状态和承诺根，然后才能转发数据包。两条链直接的中继器进程负责频繁查询两条链的状态，以确定何时必须中继什么。

### 捆绑

如果主机状态机支持，则中继器进程可以将许多数据报捆绑到一个交易中，这将导致它们按顺序执行，并平摊所有开销成本（例如，签名检查费用）。

### 竞态条件

在同一对模块和链之间中继的多个中继器可能会尝试同时中继相同的数据包（或提交相同的区块头）。如果两个中继器这样做，第一个交易将成功，第二个交易将失败。中继器之间或发送原始数据包的参与者与中继器之间的带外协调对于缓解这种情况是必要的。进一步的讨论超出了本标准的范围。

### 激励措施

中继进程必须能够访问两条链上的账户，并具有足够的余额来支付交易费用。中继器可以使用应用程序级别的方法来收回这些费用，例如通过在数据包数据中包含对自己的小额费用——中继器费用支付协议将在此 ICS 的未来版本或单独的 ICS 中描述。

可以安全的并行运行任意数量的中继器进程（实际上，预计单独的中继器会服务于链间的单独子集）。但是，如果他们多次提交相同的证明，则可能会花费不必要的费用，因此一些最小的协调可能是理想的（例如，将特定的中继器分配给特定的数据包或扫描内存池以查找未处理的交易）。

## 向后兼容性

不适用。中继器进程是链下的，可以根据需要进行升级或降级。

## 向前兼容性

不适用。中继器进程是链下的，可以根据需要进行升级或降级。

## 示例实现

即将到来。

## 其他实现

即将到来。

## 历史

2019年3月30日-提交初稿

2019年4月15日-修订格式和清晰度

2019年4月23日-注释修订；草案合并

## 版权

本规范所有内容均采用 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 许可授权。
