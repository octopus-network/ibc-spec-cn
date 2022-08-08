# IBC

![banner](https://github.com/octopus-network/ibc/blob/master/assets/interchain-standards-image.jpg)

## 概要

本仓库是跨链通信协议（IBC） 开发和文档的规范仓库。

本仓库应用于整合 IBC 的设计原理、协议语义和编码描述，包括核心传输、身份验证和排序层（IBC/TAO），以及描述数据包编码和处理语义的应用层 (IBC/APP)。

欢迎参与贡献。有关贡献指南，请参阅 [CONTRIBUTING.md](https://github.com/cosmos/ibc/blob/main/meta/CONTRIBUTING.md) 。

请参阅 [ROADMAP.md](https://github.com/cosmos/ibc/blob/main/meta/ROADMAP.md) 获取路线图最新公开版本。

## 跨链标准

处于或通过“草案”阶段的所有标准在此处按其 ICS 编号的顺序列出，并按类别排序。

### Meta

Interchain Standard Number | Standard Title | Stage
--- | --- | ---
[1](spec/ics-001-ics-standard/README.md) | ICS Specification Standard | N/A

### Core

Interchain Standard Number | Standard Title | Stage
--- | --- | ---
[2](spec/core/ics-002-client-semantics/README.md) | Client Semantics | Candidate
[3](spec/core/ics-003-connection-semantics/README.md) | Connection Semantics | Candidate
[4](spec/core/ics-004-channel-and-packet-semantics/README.md) | Channel &amp; Packet Semantics | Candidate
[5](spec/core/ics-005-port-allocation/README.md) | Port Allocation | Candidate
[23](spec/core/ics-023-vector-commitments/README.md) | Vector Commitments | Candidate
[24](spec/core/ics-024-host-requirements/README.md) | Host Requirements | Candidate
[25](spec/core/ics-025-handler-interface/README.md) | Handler Interface | Candidate
[26](spec/core/ics-026-routing-module/README.md) | Routing Module | Candidate

### Client

Interchain Standard Number | Standard Title | Stage
--- | --- | ---
[6](spec/client/ics-006-solo-machine-client/README.md) | Solo Machine Client | Candidate
[7](spec/client/ics-007-tendermint-client/README.md) | Tendermint Client | Candidate
[8](spec/client/ics-008-wasm-client/README.md) | Wasm Client | Draft
[9](spec/client/ics-009-loopback-client/README.md) | Loopback Client | Candidate
[10](spec/client/ics-010-grandpa-client/README.md) | GRANDPA Client | Draft

### Relayer

Interchain Standard Number | Standard Title | Stage
--- | --- | ---
[18](spec/relayer/ics-018-relayer-algorithms/README.md) | Relayer Algorithms | Candidate

### App

Interchain Standard Number | Standard Title | Stage
--- | --- | ---
[20](spec/app/ics-020-fungible-token-transfer/README.md) | Fungible Token Transfer | Candidate
[27](spec/app/ics-027-interchain-accounts/README.md) | Interchain Accounts | Draft
[29](spec/app/ics-029-fee-payment) | General Relayer Incentivisation Mechanism | Candidate
[30](spec/app/ics-030-middleware) | IBC Application Middleware | Candidate

## 中文译者

* [Julian](https://github.com/en) - <julian@oct.network>
* [Kim](https://github.com/emmkim404) - <630467286@qq.com>
* [Burt](https://github.com/livelybug) - <burt@oct.network>
* [Arthas](https://github.com/arthaszeng) - <sli616@163.com>
* [Jeffrey](https://github.com/hulatown) - <huzhiwei@outlook.com>
* [Alex](https://github.com/alexwanng) - <idea.wangtj@gmail.com>
* [Evie](https://github.com/eviexu2) - <evie@bianjie.ai>
* [John](https://github.com/Satoshi-Kusumoto)
* [Vincent](https://github.com/chengwenxi) - <vincent.ch.cn@gmail.com>
* [Leo](https://github.com/leoyoung0)
* [Lester](https://github.com/lesterli) - <lester@oct.network>
* [xiangjianmeng](https://github.com/xiangjianmeng) - <mengxiangjian888@Gmail.com>
* [Louis](https://github.com/toliuyi) - <louis@oct.network>
