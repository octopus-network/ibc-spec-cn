---
ics: '5'
title: 端口分配
stage: 草案
requires: '24'
required-by: '4'
category: IBC/TAO
kind: 接口
author: Christopher Goes <cwgoes@tendermint.com>
created: '2019-06-20'
modified: '2019-08-25'
---

## 概要

该标准指定了端口分配系统，模块可以通过该系统绑定到由 IBC 处理程序分配的唯一命名的端口。 然后可以将端口用于创建通道，并且可以被最初绑定到端口的模块转移或释放。

### 动机

区块链间通信协议旨在促进模块之间的通信，其中模块是独立的，可能相互不信任，在自治账本上执行的自成一体的代码。为了提供所需的端到端语义，IBC 处理程序必须实现对特定模块许可的通道。 该规范定义了实现该模型的*端口分配和所有权*系统。

可能会出现关于哪种模块逻辑可以绑定到特定端口名称的约定，例如“bank”用于处理同质通证，“staking”用于链间抵押的。 这类似于 HTTP 服务器的 80 端口的惯用用法——该协议无法强制将特定的模块逻辑实际上绑定到惯用端口，因此用户必须自己检查。可以创建具有伪随机标识符的临时端口以用于临时协议处理。

模块可以绑定到多个端口，并连接到不同的计算机上另一个模块绑定的多个端口。任何数量的（唯一标识的）通道都可以同时使用一个端口。通道是在两个端口之间的端到端的，每个端口必须事先已被模块绑定，然后模块将控制该通道的一端。

（可选）主机状态机可以选择将端口绑定通过生成专门用于绑定端口的能力键的方式暴露给特别允许的模块管理器 。然后模块管理器可以使用自定义规则集控制模块可以绑定到哪些端口，和仅被管理器验证后端口名称和模块后才转移端口到其他模块。路由模块可以扮演这个角色（请参阅 [ICS 26](../ics-026-routing-module) ）。

### 定义

`Identifier` ， `get` ， `set`和`delete`的定义与 [ICS 24](../ics-024-host-requirements) 中的相同。

*端口*是一种特殊的标识符，用于许可模块创建和使用通道。

*模块*是主机状态机的子组件，独立于 IBC 处理程序。例如以太坊智能合约和  Cosmos SDK 和 Substrate 的模块。 除了主机状态机可以使用对象能力或源身份验证来访问模块的许可端口的能力之外，IBC 规范不对模块功能进行任何假设。

### 所需属性

- 一个模块绑定到端口后，其他模块将无法使用该端口，直到该模块释放它
- 一个模块可以选择释放端口或将其转移到另一个模块
- 单个模块可以一次绑定到多个端口
- 分配端口时，先到先得，先绑定先服务，链可以在第一次启动时将已知模块绑定“保留”端口。

作为一个有帮助的比较，以下 TCP 的类比大致准确：

IBC 概念 | TCP/IP 概念 | 差异性
--- | --- | ---
IBC | TCP | 很多，请参阅描述 IBC 的体系结构文档
端口（例如“bank”） | 端口（例如 80） | 没有低位数字的保留端口，端口为字符串
模块（例如“bank”） | 应用程序（例如 Nginx） | 特定于应用
客户端 | - | 没有直接的类比，有点像 L2 路由，也有点像 TLS
连接 | - | 没有直接的类比，合并进了 TCP 的连接
通道 | 连接 | 可以同时打开或关闭任意数量的通道

## 技术规范

### 数据结构

主机状态机务必支持对象能力引用或模块的源认证。

在前一种支持对象能力的情况下，IBC 处理程序必须支持生成唯一的，不透明的*对象能力*引用的能力，它可以传递给某个模块，而其他模块则无法复制。两个示例是 Cosmos SDK（ [参考](https://github.com/cosmos/cosmos-sdk/blob/97eac176a5d533838333f7212cbbd79beb0754bc/store/types/store.go#L275) ）中使用的存储密钥和 Agoric 的 Javascript 运行时中使用的对象引用（ [参考](https://github.com/Agoric/SwingSet) ）。

```typescript
type CapabilityKey object
```

`newCapability`必须使用一个名称生成唯一的能力键，这样该名称将本地映射到能力键，并且以后可以与`getCapability`一起使用。

```typescript
function newCapability(name: string): CapabilityKey {
  // 由主状态机提供, 例如 Cosmos SDK 的 ADR 3 / ScopedCapabilityKeeper
}
```

`authenticateCapability`必须接受一个名称和能力键，并检查名称是否在本地映射到所提供的能力。该名称可以是不受信任的用户输入。

```typescript
function authenticateCapability(name: string, capability: CapabilityKey): bool {
  // 由主状态机提供, 例如 Cosmos SDK 的 ADR 3 / ScopedCapabilityKeeper
}
```

`claimCapability`必须接受一个名称和能力键（由另一个模块提供），并将其本地映射到能力，“声明”该能力以备将来使用。

```typescript
function claimCapability(name: string, capability: CapabilityKey) {
  // // 由主状态机提供, 例如 ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

`getCapability`必须允许模块查找其先前创建的能力或按名称声明的能力。

```typescript
function getCapability(name: string): CapabilityKey {
  // 由主状态机提供, 例如 Cosmos SDK 的 ADR 3 / ScopedCapabilityKeeper
}
```

`releaseCapability`必须允许模块释放其拥有的能力。

```typescript
function releaseCapability(capability: CapabilityKey) {
  // 由主状态机提供, 例如 Cosmos SDK 的 ADR 3 / ScopedCapabilityKeeper
}
```

在后一种源身份验证的情况下，IBC 处理程序必须具有安全读取调用模块的*源标识符*的能力， 主机状态机中每个模块的唯一字符串，不能由该模块更改或由另一个模块伪造。 一个示例是以太坊（ [参考](https://ethereum.github.io/yellowpaper/paper.pdf) ）使用的智能合约地址。

```typescript
type SourceIdentifier string
```

```typescript
function callingModuleIdentifier(): SourceIdentifier {
  // 由主状态机提供, 例如 以太坊中的合约地址
}
```

然后按以下方式实现`newCapability` ， `authenticateCapability` ， `claimCapability` ， `getCapability`和`releaseCapability` ：

```typescript
function newCapability(name: string): CapabilityKey {
  return callingModuleIdentifier()
}
```

```typescript
function authenticateCapability(name: string, capability: CapabilityKey) {
  return callingModuleIdentifier() === name
}
```

```typescript
function claimCapability(name: string, capability: CapabilityKey) {
  // 无操作
}
```

```typescript
function getCapability(name: string): CapabilityKey {
  // 实际没有用到
  return nil
}
```

```typescript
function releaseCapability(capability: CapabilityKey) {
  // 无操作
}
```

#### 储存路径

`portPath`接受一个`Identifier`参数并返回存储路径，在该路径下应存储与端口关联的对象能力引用或所有者模块标识符。

```typescript
function portPath(id: Identifier): Path {
    return "ports/{id}"
}
```

### 子协议

#### 标识符验证

端口的所有者模块标识符存储在唯一的`Identifier`前缀下。 可以提供验证函数`validatePortIdentifier` 。

```typescript
type validatePortIdentifier = (id: Identifier) => boolean
```

如果未提供，默认的`validatePortIdentifier`函数将始终返回`true` 。

#### 绑定到端口

IBC 处理程序必须实现`bindPort` 。 `bindPort`绑定到未分配的端口，如果该端口已被分配，则绑定失败。

如果主机状态机未实现特殊的模块管理器来控制端口分配，则`bindPort`应该对所有模块都可用。否则`bindPort`应该只能由模块管理器调用。

```typescript
function bindPort(id: Identifier): CapabilityKey {
    abortTransactionUnless(validatePortIdentifier(id))
    abortTransactionUnless(getCapability(portPath(id)) === null)
    capability = newCapability(portPath(id))
    return capability
}
```

#### 转让端口所有权

如果主机状态机支持对象能力，则不需要增加此协议，因为端口的引用承载了能力。

#### 释放端口

IBC 处理程序必须实现`releasePort`函数，该函数允许模块释放端口，以便其他模块随后可以绑定到该端口。

`releasePort`应对所有模块均可用。

> 警告：释放端口将允许其他模块绑定到该端口，并可能拦截传入的通道创建握手请求。仅在安全的情况下，模块才应释放端口。

```typescript
function releasePort(capability: CapabilityKey) {
    abortTransactionUnless(authenticateCapability(portPath(id), capability))
    releaseCapability(capability)
}
```

### 属性与不变性

- 默认情况下，端口标识符是遵循“先到先服务”原则的：模块绑定到端口后，只有该模块才能使用该端口，直到模块转移或释放它为止。模块管理器可以实现自定义逻辑，以覆盖此逻辑。

## 向后兼容性

不适用。

## 向前兼容性

端口绑定不是线路协议（wire protocol），因此只要所有权语义不受影响，接口就可以在单独的链上独立更改。

## 示例实现

即将到来。

## 其他实现

即将到来。

## 历史

2019年6月29日-初稿

## 版权

本规范所有内容均采用 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 许可授权。
