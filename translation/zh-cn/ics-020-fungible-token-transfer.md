---
ics: 20
title: Fungible Token Transfer
stage: draft
category: IBC/APP
requires: 25, 26
kind: instantiation
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-07-15 
modified: 2019-08-25
---

## Synopsis / 概览
This standard document specifies packet data structure, state machine handling logic, and encoding details for the transfer of fungible tokens over an IBC channel between two modules on separate chains. The state machine logic presented allows for safe multi-chain denomination handling with permissionless channel opening. This logic constitutes a "fungible token transfer bridge module", interfacing between the IBC routing module and an existing asset tracking module on the host state machine.

该标准规定了通过IBC通道(channel)在各自链上的两个模块之间进行代币转移的`packet`数据结构，状态机处理逻辑以及编码细节。本文所描述的状态机逻辑允许在无许可通道(channel)打开的情况下安全地处理多个链的代币。该逻辑通过在节点状态机上的IBC路由模块和一个现存的资产跟踪模块之间建立连接实现了一个可互换代币转移的桥接模块。

### Motivation / 动机
Users of a set of chains connected over the IBC protocol might wish to utilise an asset issued on one chain on another chain, perhaps to make use of additional features such as exchange or privacy protection, while retaining fungibility with the original asset on the issuing chain. This application-layer standard describes a protocol for transferring fungible tokens between chains connected with IBC which preserves asset fungibility, preserves asset ownership, limits the impact of Byzantine faults, and requires no additional permissioning.

基于IBC协议连接的一组链的用户可能希望在一条链上能利用在另一条链上发行的资产来使用该链上的附加功能，例如交易所或隐私保护，同时保持和发行链上的原始资产的互换性。该应用层标椎描述了一个用于在基于IBC连接的链间转移可互换代币的协议，该协议保留了资产的可互换性和资产所有权，限制了拜占庭错误的影响，并且无需额外许可。

### Definitions / 定义
The IBC handler interface & IBC routing module interface are as defined in [ICS 25](../ics-025-handler-interface) and [ICS 26](../ics-026-routing-module), respectively.

[ICS 25](../ics-025-handler-interface) 和 [ICS 26](../ics-026-routing-module)分别定义了IBC处理接口和IBC路由模块接口

### Desired Properties / 所需属性
- Preservation of fungibility (two-way peg).
- 保持可互换性（双向锚定）。
- Preservation of total supply (constant or inflationary on a single source chain & module).
- 保持总量不变（在单一源链和模块上保持不变或通胀）。
- Permissionless token transfers, no need to whitelist connections, modules, or denominations.
- 无许可的代币转移，无需将连接（connections）、模块或代币面额加入白名单。
- Symmetric (all chains implement the same logic, no in-protocol differentiation of hubs & zones).
- 对称（所有链实现相同的逻辑，hubs和zones无协议差别）。
- Fault containment: prevents Byzantine-inflation of tokens originating on chain `A`, as a result of chain `B`'s Byzantine behaviour (though any users who sent tokens to chain `B` may be at risk).
- 容错：防止由于链`B`的拜占庭行为造成源自链`A`的代币的拜占庭通货膨胀（尽管任何将代币转移到链`B`上的用户都面临风险）。

## Technical Specification / 技术规范

### Data Structures / 数据结构
Only one packet data type, `FungibleTokenPacketData`, which specifies the denomination, amount, sending account, receiving account, and whether the sending chain is the source of the asset, is required.

仅需要一个packet数据类型`FungibleTokenPacketData`，该类型指定了面额，数量，发送账户，接受账户以及发送链是否为资产的发行链。

```typescript
interface FungibleTokenPacketData {
  denomination: string
  amount: uint256
  sender: string
  receiver: string
  source: boolean
}
```

The fungible token transfer bridge module tracks escrow addresses associated with particular channels in state.Fields of the `ModuleState` are assumed to be in scope.
可互换代币转移桥接模块跟踪与状态中指定通道关联的托管地址。假设`ModuleState`的字段在范围内。

```typescript
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### Sub-protocols / 子协议
The sub-protocols described herein should be implemented in a "fungible token transfer bridge" module with access to a bank module and to the IBC routing module.

可以访问bank模块和IBC路由模块的“可互换代币转移桥接”模块中实现了子协议。

#### Port & channel setup / 端口 & 通道设置
The `setup` function must be called exactly once when the module is created (perhaps when the blockchain itself is initialised) to bind to the appropriate port and create an escrow address (owned by the module).

当创建“可互换代币转移桥接”模块时（也可能是区块链本身初始化时），必须仅调用一次`setup`函数用于绑定到对应的端口并创建一个托管地址（该地址由模块所有）。

```typescript
function setup() {
  routingModule.bindPort("bank", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
}
```

Once the `setup` function has been called, channels can be created through the IBC routing module between instances of the fungible token transfer module on separate chains.
调用`setup`函数后，通过在不同链上的可互换代币转移模块之间的IBC路由模块创建通道。

An administrator (with the permissions to create connections & channels on the host state machine) is responsible for setting up connections to other state machines & creating channels to other instances of this module (or another module supporting this interface) on other chains. This specification defines packet handling semantics only, and defines them in such a fashion
that the module itself doesn't need to worry about what connections or channels might or might not exist at any point in time.
管理员（具有在节点的状态机上创建连接和通道的权限）负责在本地链与其他链的状态机之间创建连接，在本地链与其他链的该模块（或支持该接口的其他模块）的实例之间通道。本规范仅定义了packet处理语义，以及模块本身在任意时间点都无需关心连接或通道是否存在。

#### Routing module callbacks / 路由模块回调

##### Channel lifecycle management / 通道生命周期管理

Both machines `A` and `B` accept new channels from any module on another machine, if and only if:
机器A和机器B在当且仅当在以下情况下接受来自第三台机器上任何模块的新通道创建请求：

- The other module is bound to the "bank" port.
- 第三台机器的模块绑定到“bank”端口。
- The channel being created is unordered.
- 创建的通道是无序的。
- The version string is empty.
- 版本号为空。

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // version not used at present
  abortTransactionUnless(version === "")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // version not used at present
  abortTransactionUnless(version === "")
  abortTransactionUnless(counterpartyVersion === "")
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // version not used at present
  abortTransactionUnless(version === "")
  // port has already been validated
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

##### Packet relay / 数据包中继

In plain English, between chains `A` and `B`:
简单来说，在`A` 和 `B`两个链间：

- When acting as the source zone, the bridge module escrows an existing local asset denomination on the sending chain and mints vouchers on the receiving chain.
- 在源zone上，桥接模块会在发送链上托管现有的本地资产面额，并在接收链上生成凭证。
- When acting as the sink zone, the bridge module burns local vouchers on the sending chains and unescrows the local asset denomination on the receiving chain.
- 在目标zone上，桥接模块会在发送链上刻录本地凭证，并在接收链上解除对本地资产面额的托管。
- When a packet times-out, local assets are unescrowed back to the sender or vouchers minted back to the sender appropriately.
- 当数据包超时时，本地资产将解除托管并退还给发送者，或将凭证发回给发送者。
- No acknowledgement data is necessary.
- 无需数据确认。

`createOutgoingPacket` must be called by a transaction handler in the module which performs appropriate signature checks, specific to the account owner on the host state machine.
模块中对节点状态机上的账户所有者进行签名检查的交易处理程序必须调用`createOutgoingPacket`。

```typescript
function createOutgoingPacket(
  denomination: string,
  amount: uint256,
  sender: string,
  receiver: string,
  source: boolean) {
  if source {
    // sender is source chain: escrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.sourceChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/destPort}/{packet.destChannel}"
    abortTransactionUnless(denomination.slice(0, len(prefix)) === prefix)
    // escrow source tokens (assumed to fail if balance insufficient)
    bank.TransferCoins(sender, escrowAccount, denomination.slice(len(prefix)), amount)
  } else {
    // receiver is source chain, burn vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(denomination.slice(0, len(prefix)) === prefix)
    // burn vouchers (assumed to fail if balance insufficient)
    bank.BurnCoins(sender, denomination, amount)
  }
  FungibleTokenPacketData data = FungibleTokenPacketData{denomination, amount, sender, receiver, source}
  handler.sendPacket(packet)
}
```

`onRecvPacket` is called by the routing module when a packet addressed to this module has been received.
当路由模块收到一个数据包后调用`onRecvPacket`。

```typescript
function onRecvPacket(packet: Packet): bytes {
  FungibleTokenPacketData data = packet.data
  if data.source {
    // sender was source chain: mint vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/destPort}/{packet.destChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // mint vouchers to receiver (assumed to fail if balance insufficient)
    bank.MintCoins(data.receiver, data.denomination, data.amount)
  } else {
    // receiver is source chain: unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // unescrow tokens to receiver (assumed to fail if balance insufficient)
    bank.TransferCoins(escrowAccount, data.receiver, data.denomination.slice(len(prefix)), data.amount)
  }
  return 0x
}
```

`onAcknowledgePacket` is called by the routing module when a packet sent by this module has been acknowledged.
当由路由模块发送的数据包被确认后，该模块调用`onAcknowledgePacket`。

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // nothing is necessary, likely this will never be called since it's a no-op
}
```

`onTimeoutPacket` is called by the routing module when a packet sent by this module has timed-out (such that it will not be received on the destination chain).
当由路由模块发送的数据包超时（例如数据包没有被目标链接收到）后，路由模块调用`onTimeoutPacket`。

```typescript
function onTimeoutPacket(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  if data.source {
    // sender was source chain, unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // unescrow tokens back to sender
    bank.TransferCoins(escrowAccount, data.sender, data.denomination.slice(len(prefix)), data.amount)
  } else {
    // receiver was source chain, mint vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // mint vouchers back to sender
    bank.MintCoins(data.sender, data.denomination, data.amount)
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // can't happen, only unordered channels allowed
}
```

#### Reasoning / 原理

##### Correctness / 正确性

This implementation preserves both fungibility & supply.
该实现保持了可互换性和总量不变。

Fungibility: If tokens have been sent to the counterparty chain, they can be redeemed back in the same denomination & amount on the source chain.
可互换性：如果代币已发送到目标链，则可以以相同面额和数量兑换回源链。

Supply: Redefine supply as unlocked tokens. All send-recv pairs sum to net zero. Source chain can change supply.
总量：将供应重新定义为未锁定的代币。所有源链的发送量等于目标链的接受量。源链可以改变代币的总量。

##### Multi-chain notes / 多链注意事项

This does not yet handle the "diamond problem", where a user sends a token originating on chain A to chain B, then to chain D, and wants to return it through D -> C -> A — since the supply is tracked as owned by chain B, chain C cannot serve as the intermediary. It is not yet clear whether that case should be dealt with in-protocol or not — it may be fine to just require the original path of redemption (and if there is frequent liquidity and some surplus on both paths the diamond path will work most of the time). Complexities arising from long redemption paths may lead to the emergence of central chains in the network topology.
还无法处理“菱形问题”，即用户将在链A上发行的代币跨链转移到链B，然后又转移到链D，并想通过D -> C -> A的路径将代币转移回链A，由于此时代币的总量被认为是由链B控制，链C无法作为中介。目前尚不确定该场景是否应该在协议内处理，可能只需原路返回即可（如果两条路径上都有频繁的流动性和结余，菱形路径会更有效）。长的赎回路径产生的复杂性会导致网络拓扑中中心链的出现。

#### Optional addenda / 可选附录

- Each chain, locally, could elect to keep a lookup table to use short, user-friendly local denominations in state which are translated to and from the longer denominations when sending and receiving packets. 
- 每个链可以选择在本地维护一个查找表在状态中使用短的，用户友好的本地面额。当发送数据包时这些面额会被转换成较长的面额和接受数据包时较长的面额会被转换成这些面额。
- Additional restrictions may be imposed on which other machines may be connected to & which channels may be established.
- 可能对哪些计算机可以被连接以及哪些通道可以建立会有额外的限制。

## Backwards Compatibility / 后向兼容
Not applicable.
不适用。

## Forwards Compatibility / 前向兼容
A future version of this standard could use a different version in the channel handshake.

该标准的未来版本可能使用不同的通道创建版本。

## Example Implementation / 样例实现
Coming soon.

即将到来。

## Other Implementations / 其他实现
Coming soon.

即将到来。

## History / 历史
Jul 15, 2019 - Draft written

2019年7月15 - 草案完成

Jul 29, 2019 - Major revisions; cleanup

2019年7月29 - 主要修订；整理

Aug 25, 2019 - Major revisions, more cleanup

2019年8月25 - 主要修订；进一步整理

## Copyright / 版权
All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).

本文所有内容遵循 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0)协议。
