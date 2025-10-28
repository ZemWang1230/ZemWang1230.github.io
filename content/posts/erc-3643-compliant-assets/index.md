---
date: '2025-10-26T17:46:44+08:00'
title: 'ERC-3643:链上合规资产'
author: ["Zem"]
summary: "An institutional grade security token contract that provides interfaces for the management and compliant transfer of security tokens."
tags: ["ERC-3643", "T-REX", "RWA", "ERC-20"]
series: ["EIP学习笔记"]
---

### ERC-3643的来历与意义

> ERC-3643是一个面向 “**受监管／许可型 Token ”** 的智能合约标准，用以实现资产（尤其是真实资产 Real-World Assets、证券、债券、基金份额等）上链并同时附带合规、身份验证、转让限制等机制。

传统的ERC20可以自由转让，只要余额足够；而ERC3643在Token的生命周期里内嵌了合规控制，也就是只有满足预设的身份、资格、地域等条件的钱包才能持有或者转让；

### 核心思想

ERC-3643在ERC-20的基础上引入了三个抽象层：

1. 身份层

` IdentityRegistry ` / ` ClaimTopicsRegistry ` / ` TrustedIssuersRegistry `

该层管理用户的身份与声明；

2. 合规层

` Compliance `

该层定义并执行合规检查逻辑；

3. 资产层

` SecurityToken `

实际代币合约（继承ERC-20）；

整个体系像这样运作（每次转账，都会在背后触发 "身份验证" + "合规检查" 双重校验）：

```solidity
用户 ──► IdentityRegistry ──► Compliance ──► Token.transfer()
```

### 核心组件

ONCHAINID 是“身份证”，Trusted Issuers / Claim Topics 是“发证机关白名单”，Identity Registry 是“安检”，Compliance 是“法规”，Security Token 是“资产本身”。六者解耦，才做到“合规但可升级”的证券级代币。

**1. ONCHAINID（去中心化身份）**

https://github.com/onchain-id/solidity

该部分是由用户自己部署的合约，用于与证劵型代币或任何其他可能需要链上身份的应用进行交互，存储与特定身份相关的密钥与声明；

**2. Identity Registry（身份注册表）**

负责维护所有参与者的身份状态，需要每个地址**必须**绑定一个` ONCHAINID `合约地址，并负责声明校验；

它建立了钱包地址、Identity合约、投资者国家之间的连接；

有以下核心函数：

```solidity
// 注册身份，将用户地址与身份地址合约关联起来，可以理解为链上登记户口；
function registerIdentity(address _userAddress, IIdentity _identity, uint16 _country) external;
// 根据用户Identity合约中声明的有效性（根据证劵型代币要求）返回一个状态；
function isVerified(address _userAddress) external view returns (bool);
```

**3. Claim Topics Registry（声明主题注册表）**

存储所有可信声明主题类型，比如KYC（Know Your Customer）、AML（Anti-Money Laundering）等；

有以下核心函数：

```solidity
// 向“信任声明主题白名单”里追加一个主题编号，比如KYC=1, AML=2;
function addClaimTopic(uint256 _claimTopic) external;
```

**4. Trusted Issuers Registry（可信发行者注册表）**

存储所有可信声明发行者` IClaimIssuer `的合约地址；

声明与维护哪些机构/地址可以签发这些声明，投资者的` Identity `合约必须由存储在此的合约中的声明发行者` IClaimIssuer `签名的声明，才能持有代币；

有以下核心函数：

```solidity
// 将一个ClaimIssuer合约注册为可信任的声明发行方，同时分配一批声明主题;
function addTrustedIssuer(IClaimIssuer _trustedIssuer, uint256[] calldata _claimTopics) external;
```

**5. Compliance Smart Contract（合规逻辑模块）**

用于实现各种合规规则，比如：仅允许某个国家的用户交易、限制持仓的比例、限制特定的转账路径、添加锁仓期；

该部分由Token在每个交易中触发（类似callback函数），如果交易符合发行的规则，返回True；

有以下核心函数：

```solidity
// 检查交易是否合规，只读函数
function canTransfer(address _from, address _to, uint256 _amount) external view returns (bool);
```

**6. Security Token Contract（证劵型代币）**

继承自ERC-20，与身份注册表交互，以确认投资者的合格状态，从而允许其持有和交易代币；

同时支持监管方的特殊操作，例如` forcedTransfer() `、` pause() `；

有以下核心函数：

```solidity
// 在无需原持有人签名、也不受普通冻结限制的情况下，把指定数量的代币从任意地址强制转移到另一个已白名单地址;
function forcedTransfer(address _from, address _to, uint256 _amount) external returns (bool);
// 让“丢私钥”的投资者在不迁移合约、不硬分叉、不双花的前提下，把代币余额无损地搬到新的钱包地址;
function recoveryAddress(address _lostWallet, address _newWallet, address _investorOnchainID) external returns (bool);
```

### 核心结构

```bash
 ┌─────────────────────────────────────────────┐
 │               SecurityToken                 │
 │─────────────────────────────────────────────│
 │  ERC-20逻辑 + 合规检查 + 强制转移 + 事件记录    │
 └──────────────┬──────────────────────────────┘
                │
                ▼
 ┌─────────────────────────────────────────────┐
 │                 Compliance                  │
 │─────────────────────────────────────────────│
 │    规则验证 / 白名单 / 限制条件 / 可扩展插件     │
 └──────────────┬──────────────────────────────┘
                │
                ▼
 ┌─────────────────────────────────────────────┐
 │               Identity Registry             │
 │─────────────────────────────────────────────│
 │     地址 <-> OnchainID <-> Claim 验证        │
 └─────────────────────────────────────────────┘
```





















































