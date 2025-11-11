---
date: '2025-11-09T14:29:51+08:00'
title: 'T-REX:ERC-3643的参考实现'
author: ["Zem"]
summary: "The T-REX (Token for Regulated EXchanges) protocol is a comprehensive suite of Solidity smart contracts, implementing the ERC-3643 standard and designed to enable the issuance, management, and transfer of security tokens in compliance with regulations. It ensures secure and compliant transactions for all parties involved in the token exchange."
tags: ["ERC-3643", "T-REX", "RWA", "ERC-20"]
series: ["EIP学习笔记"]
---

### T-REX是什么

仓库地址：https://github.com/TokenySolutions/T-REX

T-REX（Token for Regulated EXchanges）协议是一套全面的 Solidity 智能合约套件，实现了 ERC-3643 标准，旨在以符合监管要求的方式发行、管理和转移证券型代币。该协议确保代币交换中所有参与方的交易安全且合规。

T- REX展示了身份、合规、代币三层如何协同工作；

代码目录结构：

```bash
contracts/
├── token/              # 核心代币合约
├── registry/           # 身份注册系统
├── compliance/         # 合规检查系统
├── proxy/              # 代理合约（可升级性）
├── factory/            # 工厂合约（部署套件）
├── roles/              # 角色权限管理
├── DVA/                # Delivery vs Approval 转账管理
├── DVD/                # Delivery vs Delivery 转账管理
└── _testContracts/     # 测试合约
```

### 核心模块深入解析

#### 链上去中心化身份（ONCHAINID）

一个用户自己的合约，把用户身份以智能合约的形式放在链上，主要用于把KYC/合规信息以链上可验证的方式与代币/合约交互；

1. 用户自己部署一个ONCHAINID合约（Identity.sol），地址即为链上身份；

2. KYC提供商（Claim Issuer）对用户进行线下认证KYC；

3. KYC提供商（Claim Issuer）调用` addClaim() `向用户的Identity合约写入一条声明（claim）；

4. 发行方的代币在执行转账时，会查询接收方的ONCHAINID，` IIdentity(...).getClaim() `，检查接收方是否具有特定类型的claim；

在源码中，链上身份系统的核心是由三个结构体和一组基础逻辑来组成的：` Key `、` Execution `、` Claim `，它们的各种状态单独存储在` Storage.sol `文件中，上层的` Identity.sol `和` ClaimIssuer.sol `和` Verifier.sol `等基础逻辑都依托于这些结构；

```bash
Key (谁有权操作)
   ↓ 控制执行
Execution (发起或批准交易)
   ↓ 产生或修改声明
Claim (由 Issuer 写入合规声明)
   ↓ 提供外部验证入口
Token / Compliance 模块读取 Claim → 验证身份合法性
```

**Key-身份控制单元**

```solidity
struct Key {
    uint256[] purposes; // 用途类型
    uint256 keyType;    // 密钥类型（1=ECDSA, 2=RSA...）
    bytes32 key;        // 公钥哈希
}
```

每个身份合约中都有一组 Key，代表不同用途的公钥或地址。
Key的核心是以 `bytes32` 的哈希形式保存的公钥，通过 `purpose` 实现权限分层：

- `1 = MANAGEMENT`：管理权限，可增删 Key、执行交易；
- `2 = ACTION`：执行权限，用于发起链上操作；
- `3 = CLAIM`：声明签名权限，通常由合规机构（Issuer）持有。

相关的存储：

|     存储字段     |              类型               |           说明            |                             示例                             |
| :--------------: | :-----------------------------: | :-----------------------: | :----------------------------------------------------------: |
|     `_keys`      |    `mapping(bytes32 => Key)`    | 按公钥哈希索引的 Key 集合 |                      _keys[公钥] = KEY                       |
| `_keysByPurpose` | `mapping(uint256 => bytes32[])` | 不同权限类型下的 Key 列表 | _keysByPurpose[1] = [MANAGEMENT_1,MANAGEMENT_2,...];keysByPurpose[2] = [ACTION_1,ACTION_2,...] |

相关的重要函数：

```solidity
// 添加Key，只有管理者或者合约自身能调用
function addKey(bytes32 _key, uint256 _purpose, uint256 _keyType) external returns (bool success);

// 移除Key
function removeKey(bytes32 _key, uint256 _purpose) external returns (bool success);

// 该key是否有具有MANAGEMENT或某个具体用途
function keyHasPurpose(bytes32 _key, uint256 _purpose) external view returns (bool exists);
```

Key是整个身份的访问控制基础，决定谁能管理、执行、签发声明。

**Execution-交易执行请求**

```solidity
struct Execution {
    address to; // 交互的合约地址
    uint256 value; // 交互的金额
    bytes data; // 交易的calldata
    bool approved; // 该请求的批准状态
    bool executed; // 该请求的执行状态
}
```

表示身份合约要发起的一笔交易请求，这是ERC734的核心特性，使身份本身能主动执行交易（而不仅是被动验证）。每条 Execution 会在被批准后再执行，形成一种“多签式”安全流程。

相关存储：

|     存储字段      |              类型               |                  说明                  |            示例             |
| :---------------: | :-----------------------------: | :------------------------------------: | :-------------------------: |
|   `_executions`   | `mapping(uint256 => bytes32[])` |        按请求ID索引的 请求 集合        | _executions[id] = Execution |
| `_executionNonce` |            `uint256`            | 执行请求计数器，用于唯一标识 Execution |           1,2,...           |

相关的重要函数：

```solidity
// 发起一个执行请求
// 如果sender是ACTION，并且目标地址不是identity合约，则自动批准并执行；
// 如果sender是MANAGEMENT，并且目标地址是identity合约，则自动批准并执行；
// 其余情况必须经过approve
function execute(address _to, uint256 _value, bytes calldata _data) external payable returns (uint256 executionId);

// 批准及执行一个请求（不是delegatecall，是sender直接call）
// 如果sender是ACTION，并且目标地址不是identity合约，则批准是授权的，交易会立即执行；
// 如果目标地址是identity合约，只有sender是MANAGEMENT时，批准才是授权的，交易会立即执行；
// 返回交易是否成功
function approve(uint256 _id, bool _approve) external returns (bool success);
```

Execution可以让身份合约可以安全地调用其他合约或转账。

**Claim-合规声明单元**

```solidity
struct Claim {
    uint256 topic;     // 声明类别（如KYC、居住地）
    uint256 scheme;    // 签名方案（1=ECDSA）
    address issuer;    // 签发方地址
    bytes signature;   // 签名，用于验证真实性
    bytes data;        // 声明内容或哈希
    string uri;        // 可选的链下存储引用（IPFS/HTTP等）
}
```

这是` ONCHAINID `实现 KYC/AML 等功能的核心结构，每个Claim代表由Issuer对该身份作出的一条可验证声明；

相关存储：

|     存储字段     |              类型               |           说明            |              示例              |
| :--------------: | :-----------------------------: | :-----------------------: | :----------------------------: |
|    `_claims`     |   `mapping(bytes32 => Claim)`   | 已添加的声明（Claim）集合 |    _claims[claimID] = Claim    |
| `_claimsByTopic` | `mapping(uint256 => bytes32[])` |   按类型分组的声明索引    | _claimsByTopic[国家] = claimID |

相关的重要函数：

```solidity
// 添加一个claim，需要sender具有claim权限
function addClaim(uint256 _topic, uint256 _scheme, address issuer, bytes calldata _signature, bytes calldata _data, string calldata _uri) external returns (bytes32 claimRequestId);

// 移除一个claim
function removeClaim(bytes32 _claimId) external returns (bool success);
```

Claim是身份的“合规标签”，代币或协议通过读取这些Claim判断身份是否满足转账或交互条件。

三个模块的调用链图：

```solidity
Identity.sol
 ├── uses Storage
 │    ├── _keys / _keysByPurpose → 管理权限
 │    ├── _executions / _executionNonce → 管理执行请求
 │    ├── _claims / _claimsByTopic → 管理声明
 │    └── 提供外部接口 getKey(), getClaim(), getExecution()
 └── 被 ClaimIssuer 与 Verifier 调用
```

**ClaimIssuer-声明的签发方**

Issuer通常是链上KYC或合规服务商，核心职责是：

1. 线下验证用户身份；
2. 在链上向用户的 Identity 合约写入声明；
3. 维护声明的有效性。

此部分由发行者自己部署，继承自` Identity `合约；

相关的存储：

|          存储字段           |           类型            |        说明        |            示例            |
| :-------------------------: | :-----------------------: | :----------------: | :------------------------: |
|       `revokedClaims`       | `mapping (bytes => bool)` | 存储已经撤销的声明 | revokedClaims[签名] = true |
| 及所有Identity合约的Storage |                           |                    |                            |

相关的重要函数：

```solidity
// 添加声明，需要有Claim权限
Identity.addClaim(_topic, _scheme, address(this), _signature, _data, _uri);

// 撤销一个声明，更新存储中的revokedClaims
function revokeClaim(bytes32 _claimId, address _identity) external returns(bool);

// 验证一个声明，从sig中恢复出签名者，再对签名者做hash，去验证该密钥是否属于当前ClaimIssuer合约，并且是否被赋予了CLAIM权限
function isClaimValid(IIdentity _identity, uint256 claimTopic, bytes calldata sig, bytes calldata data) external view returns (bool);
```

Issuer相当于“认证中心”，通过签名来为Identity提供可验证的身份凭证。

**Verifier-声明的验证方**

Verifier是合规检查层，负责验证身份是否拥有特定类型的合法声明；

内部存储：

|           存储字段            |                类型                |                             说明                             |
| :---------------------------: | :--------------------------------: | :----------------------------------------------------------: |
|     `requiredClaimTopics`     |            `uint256[]`             | 当前必须满足的全部主题列表，例如 `[1, 2]` 表示必须同时拥有“KYC=1”+“AML=2” 两份声明。 |
|       `trustedIssuers`        |          `IClaimIssuer[]`          |        白名单签发者地址数组，顺序无意义，仅用于遍历。        |
|  `trustedIssuerClaimTopics`   |   `mapping(address=>uint256[])`    |                 某个签发者被允许发哪些主题。                 |
| `claimTopicsToTrustedIssuers` | `mapping(uint256=>IClaimIssuer[])` |     某个主题下有哪些签发者被信任（反向索引，加速校验）。     |

内部重要函数：

```solidity
// 验证一个identity是否满足要求，如果identity有至少一个有效的claim，返回true；
function verify(address identity) public view returns(bool isVerified);
```

调用链：

```bash
ClaimIssuer.issueClaim()
      ↓
Identity.addClaim()
      ↓
Verifier.verify()
 ├─ identity.getClaim()
 └─ issuer.isClaimValid()
```

所以总结一下就是链上身份由3个角色影响：

1. **Identity（用户身份合约）**：代表个人或机构的链上身份；

2. **Claim Issuer（声明签发者）**：通常是KYC/AML服务提供商；

3. **Verifier / Token Contract（验证方）**：例如合规代币合约或DApp，在执行操作前查询Identity的合法性。

**身份 ≠ 地址，而是一个由多密钥组成、可轮换可管理的身份容器；**

总的架构图：

```bash
                 ┌──────────────────────────┐
                 │        Claim Issuer       │
                 │  (KYC/AML Provider)       │
                 │  · 线下验证用户身份        │
                 │  · 签发链上 Claim          │
                 └────────────┬──────────────┘
                              │ addClaim()
                              ▼
                 ┌──────────────────────────┐
                 │      Identity 合约         │
                 │  (由用户部署与控制)        │
                 │  · 存储声明（Claim）       │
                 │  · 允许验证者查询           │
                 └────────────┬──────────────┘
                              │ getClaim()
                              ▼
                 ┌──────────────────────────┐
                 │  Verifier / Token 合约    │
                 │  (如 ERC3643 Compliance)  │
                 │  · 校验身份合法性          │
                 │  · 控制代币交互权限         │
                 └──────────────────────────┘
```

#### 身份模块（Identity）

此部分基于ONCHAINID，负责存储链上身份和Claim，以及验证用户的Claim是否有效；核心部分在源码的` registry `目录中；

**Claim声明的管理模块（ClaimTopicsRegistry）**

将` IClaimTopicsRegistry.sol `和` CTRStorage.sol `和` ClaimTopicsRegistry.sol `结合起来看；

该模块存储必需的声明主题，告诉其他人，想要进入该套系统，必须提交这些声明才能有效；

相关的存储：

|    存储字段    |    类型     |          说明          |           示例           |
| :------------: | :---------: | :--------------------: | :----------------------: |
| `_claimTopics` | `uint256[]` | 存储所有必需的声明主题 | _claimTopics[0] = 1(KYC) |

内部重要函数：

```solidity
// 添加声明Claim，比如1=KYC、2=AML，只有owner可以调用，并且不能超过15个；
function addClaimTopic(uint256 _claimTopic) external;
```

**可信发行者的管理模块（TrustedIssuersRegistry）**

将` ITrustedIssuersRegistry.sol `和` TIRStorage.sol `和` TrustedIssuersRegistry.sol `结合起来看；

该模块存储所有的可信发行人及其他们的可信主题，系统需要决定谁（发行人）有权可以发行什么类型的声明；

相关的存储：

|            存储字段            |                 类型                 |           说明           |                         示例                         |
| :----------------------------: | :----------------------------------: | :----------------------: | :--------------------------------------------------: |
|       `_trustedIssuers`        |           `IClaimIssuer[]`           |      所有可信发行人      |              _trustedIssuers[0] = 0x..1              |
|  `_trustedIssuerClaimTopics`   |   `mapping(address => uint256[])`    | 可信发行人对应的声明主题 |      _trustedIssuerClaimTopics[0x...] = 1(KYC)       |
| `_claimTopicsToTrustedIssuers` | `mapping(uint256 => IClaimIssuer[])` | 声明主题对应的可信发行人 | _claimTopicsToTrustedIssuers[1(KYC)] = [0x..1,0x..2] |

内部重要函数：

```solidity
// 添加可信发行人及其的可信声明主题
// 可信主题不超过15个，可信发行人不能超过50个，只有owner可以调用
function addTrustedIssuer(IClaimIssuer _trustedIssuer, uint256[] calldata _claimTopics) external;

// 返回这个发行人是否有这个可信主题
function hasClaimTopic(address _issuer, uint256 _claimTopic) external view returns (bool);
```

**身份注册存储管理模块（IdentityRegistryStorage）**

将` IIdentityRegistryStorage.sol `和` IRSStorage.sol `和` IdentityRegistryStorage.sol `结合起来看；

```solidity
struct Identity {
    IIdentity identityContract;
    uint16 investorCountry;
}
```

这是身份的结构体，该部分的逻辑是得先绑定一个IdentityRegistry合约，也就是代理，然后通过代理再添加一个用户的身份合约和国家；

这样设计的目的是，确保只有可信的身份注册表合约能操作敏感的身份信息，并且数据与逻辑分离；

相关的存储：

|       存储字段        |              类型              |                 说明                 |              示例               |
| :-------------------: | :----------------------------: | :----------------------------------: | :-----------------------------: |
|     `_identities`     | `mapping(address => Identity)` |        存储用户的身份合约地址        | _identities[] = {0x..1,156(CN)} |
| `_identityRegistries` |          `address[]`           | 存储所有绑定了的IdentityRegistry地址 | _identityRegistries[0] = 0x..1  |

内部重要函数：

```solidity
// 添加一个用户的身份合约和国家
// 只有代理能调用(被包装在了IdentityRegistry.sol的registerIdentity函数中)
function addIdentityToStorage(address _userAddress, IIdentity _identity, uint16 _country) external;

// 添加一个身份注册表(IdentityRegistry合约)作为身份注册表存储(IdentityRegistryStorage合约)的代理
function bindIdentityRegistry(address _identityRegistry) external;
```

**身份注册管理模块(IdentityRegistry，最重要的模块，聚合其他模块)**

将` IIdentityRegistry.sol `和` IRStorage.sol `和` IdentityRegistry.sol `结合起来看；

这部分将相关联的模块都聚合起来，注册用户、验证用户是否通过了声明，为后面的合规及代币交易做准备；

相关的存储：

|        存储字段         |            类型            |             说明             |             示例              |
| :---------------------: | :------------------------: | :--------------------------: | :---------------------------: |
| `_tokenTopicsRegistry`  |   `IClaimTopicsRegistry`   | Claim声明管理模块的合约地址  | _tokenTopicsRegistry = 0x..1  |
| `_tokenIssuersRegistry` | `ITrustedIssuersRegistry`  | 可信发行人管理模块的合约地址 | _tokenIssuersRegistry = 0x..2 |
| `_tokenIdentityStorage` | `IIdentityRegistryStorage` |  身份注册管理模块的合约地址  | _tokenIdentityStorage = 0x..3 |

内部重要函数：

```solidity
// 注册用户的身份合约，只有代理能够调用
// 调用IdentityRegistryStorage的addIdentityToStorage
function registerIdentity(address _userAddress, IIdentity _identity, uint16 _country) external;

// 检查一个用户地址是否在身份注册表中，并且是否通过了声明
// 获取所有可信发行人->生成唯一ClaimID->拿到用户身份合约中已经颁发的声明->使用issuer.isClaimValid来验证声明是否有效
// 验证的时候为什么两步（获取及验证）：因为之前颁发的声明可能已经过期及更改
function isVerified(address _userAddress) external view returns (bool);
```

**完整的流程**

1. 部署所有组件（` ClaimTopicsRegistry `、` TrustedIssuersRegistry `、` IdentityRegistryStorage `）；
2. 部署身份注册管理合约（` IdentityRegistry `）及可信发行者(` ClaimIssuer `)；
3. 配置必需的Claim Topics(声明主题)，` claimTopicsRegistry.**addClaimTopic**(1); // KYC `；
4. 添加可信发行者，` trustedIssuersRegistry.addTrustedIssuer `；
5. 为投资者创建ONCHAINID(` deploy Identity(investor.address) `)，及颁发声明(` claimIssuer.addClaim(investorIdentity.address, 1, ...); // 添加KYC声明 `)；
6. 注册投资者(` identityRegistry.registerIdentity(investor.address, investorIdentity.address, 156); `)；
7. 验证声明是否有效(` identityRegistry.isVerified(investor.address) `)；

#### 合规模块（Compliance）

此部分定义代币交互的规则检查逻辑；核心部分在源码的` compliance `目录中；

在` T-REX `中，**合规逻辑不是硬编码在代币合约里**，而是由一个可扩展的**模块化系统（Modular Compliance）**来实现；

也就是一组可插拔的合规规则模块+核心合规协调器来实现的；

> 可以看到` compliance `目录下有两个文件夹，一个是` legacy `，一个是` modular `，这是两种架构范式的并存；
>
> `modular`是“合规即插件”架构，规则可像乐高一样自由拼装；
>
> `legacy`是“一站式合规”架构，规则在部署时固化，后期改动代价大；
>
> 两者共存是为了让发行方在“灵活/未来可扩展”与“简单/快速上线”之间做权衡。

开发者可以通过继承`IModule`接口或`AbstractModule.sol`来创建自己的合规模块，然后管理员可以通过` addModule `来将该模块绑定到合规合约中；

**模块化合规核心架构**

重点内容在` IModule.sol `和` IModularCompliance.sol `和` MCStorage.sol `和` ModularCompliance.sol `；

相关存储：

|    存储字段    |            类型            |            说明             |            示例            |
| :------------: | :------------------------: | :-------------------------: | :------------------------: |
| `_tokenBound`  |         `address`          | 合规合约绑定的Token合约地址 |    _tokenBound = 0x..1     |
|   `_modules`   |        `address[]`         |   合规合约绑定的所有模块    |    _modules[0] = 0x..2     |
| `_moduleBound` | `mapping(address => bool)` |       模块的绑定状态        | _moduleBound[0x..2] = true |

内部重要函数：

```solidity
// 绑定Token合约，只能被owner调用
function bindToken(address _token) external;

// 添加模块到合规合约，只能被owner调用，不能超过25个
function addModule(address _module) external;

// 转账时调用的函数，只能被Token合约调用
// 内部会调用IModule(_modules[i]).moduleTransferAction
function transferred(address _from, address _to, uint256 _amount) external;
// 同样的还有铸造时、销毁时的函数
// function created(address _to, uint256 _amount) external;
// function destroyed(address _from, uint256 _amount) external;

// 调用已经绑定的模块的函数
// 只能被owner调用
function callModuleFunction(bytes calldata callData, address _module) external;

// 检查转账是否合规
// 内部调用了IModule(_modules[i]).moduleCheck
function canTransfer(address _from, address _to, uint256 _amount) external view returns (bool);
```

可以看出来，` ModularCompliance.sol `通过**模块化管理**的方式，将合规逻辑从代币合约中解耦，使得合规规则可以动态添加、移除和配置，而无需修改代币合约本身。

这部分是**代币合规层的中间件**，位于**代币合约**和**合规规则模块**之间：

```bash
Token Contract
      ↓ (calls)
ModularCompliance
      ↓ (calls)
Compliance Modules (e.g., KYC, AML, Transfer Restrictions)
```

**合规模块**

这边的合约都必需实现**IModule**的接口(` moduleTransferAction `、` moduleCheck `、` isPlugAndPlay `等)，以完善合规的流程；

1. ` CountryAllowModule.sol `：国家白名单合约，有一个` _allowedCountries `存储记录着白名单，比如` _allowedCountries[msg.sender][156(CN)] = true `；

2. ` CountryRestrictModule.sol `：国家黑名单合约，有一个` _restrictedCountries `记录着黑名单；

3. ` SupplyLimitModule.sol `：总供应量限制，` _supplyLimits `记录每个合规合约的总供应量，检查时会通过` (IToken(IModularCompliance(_compliance).getTokenBound()).totalSupply() + _value) `来判断是否合规，并且是在` moduleCheck() `中检查的(不是那些Action函数)，因为Action那边是需要做完操作后更新storage的，而moduleCheck不需要，它是一个无需状态更新的全局不变式，依赖代币合约的`totalSupply()`作为权威数据源，避免冗余写入和状态漂移；
4. ` TransferFeesModule.sol `：转账费用，费用直接从转账金额里**强制划走**，收币方到账金额 = 原金额 - fee，并且需要确保模块地址已被代币设为Agent，不然模块就无权调用；
5. ...

#### 代币模块（SecurityToken）

实现受合规约束的ERC-20代币（可带冻结、白名单、转让控制等特性）；核心部分在源码的` token `目录当中；

新增的功能有：暂停代币功能、冻结代币功能、强制操作功能、恢复用户钱包功能（用户私钥丢失，想换个新钱包）；

此外，每次转账、铸造、销毁后，都会调用合规模块中相应的状态更新函数；

相关的存储：

|         存储字段         |                       类型                        |      说明      |               示例               |
| :----------------------: | :-----------------------------------------------: | :------------: | :------------------------------: |
|       `_balances`        |           `mapping(address => uint256)`           |      余额      |          _balances = 10          |
|      `_allowances`       | `mapping(address => mapping(address => uint256))` |    授权余额    | `_allowances[0x..1][0x..2] = 10` |
|      `_totalSupply`      |                     `uint256`                     |    总供应量    |        _totalSupply = 10         |
|       `_tokenName`       |                     `string`                      |    代币名字    |        _tokenName = "XXX"        |
|      `_tokenSymbol`      |                     `string`                      |    代币符号    |       _tokenSymbol = "XXX"       |
|     `_tokenDecimals`     |                      `uint8`                      |   代币小数位   |       _tokenDecimals = 18        |
|    `_tokenOnchainID`     |                     `address`                     | 代币链上身份ID |     _tokenOnchainID = 0x..2      |
|        `_frozen`         |            `mapping(address => bool)`             |    完全冻结    |      _frozen[0x..1] = true       |
|     `_frozenTokens`      |           `mapping(address => uint256)`           | 部分冻结的金额 |    _frozenTokens[0x..1] = 10     |
|      `_tokenPaused`      |                      `bool`                       |    是否暂停    |       _tokenPaused = false       |
| `_tokenIdentityRegistry` |                `IIdentityRegistry`                |  身份注册合约  |  _tokenIdentityRegistry = 0x..1  |
|    `_tokenCompliance`    |               `IModularCompliance`                |    合规合约    |     _tokenCompliance = 0x..1     |

内部重要函数：

```solidity
// 设置代币的链上身份
function setOnchainID(address _onchainID) external;

// 设置地址冻结状态
function setAddressFrozen(address _userAddress, bool _freeze) external;

// 冻结指定地址的指定数量代币
function freezePartialTokens(address _userAddress, uint256 _amount) external;

// 转账函数
// 在转账前会检查是否合规（身份注册表及合规模块均会检查一遍）
// _tokenIdentityRegistry.isVerified(_to) && _tokenCompliance.canTransfer(_from, _to, _amount)
// 转账完后，还会调用_tokenCompliance.transferred(msg.sender, _to, _amount);更新模块的状态
function transfer(address _to, uint256 _amount) public returns (bool);

// 强制在两个白名单地址之间转账
// 转账前只需要检查身份是否验证通过，_tokenIdentityRegistry.isVerified(_to)
// 情况一：from的未冻结余额足够amount，直接转账
// 情况二：from的未冻结余额不够amount，但是总的余额大于amount，先解冻超出部分的数量，再转账，转后，from里全剩冻结部分
function forcedTransfer(address _from, address _to, uint256 _amount) external returns (bool);

// 将投资者丢失的钱包中的代币强制转移到新钱包
// 首先根据新钱包生成key，去查询用户的ONCHAINID上是否有这个key及对应权限
// 随后在身份注册表中注册新钱包，强制转移资产到新钱包
// 如果之前有冻结资产，把这部分数量的资产继续冻结
// 如果之前的地址是冻结地址，将新地址也冻结
// 删除之前旧地址的身份
function recoveryAddress(address _lostWallet, address _newWallet, address _investorOnchainID) external returns (bool);
```

#### 其余模块

**权限管理系统（Roles）**

该部分实现了T-REX系统的权限管理基础设施，采用基于角色的访问控制（RBAC）模式，将系统权限划分为两大类：**Owner**角色和**Agent**角色。

Agent角色是T-REX系统中最基础的操作权限，用于执行日常管理任务。Agent权限划分为7个子角色：

| 角色名称 | 描述 | 主要功能 |
|:-------:|:----:|:-------:|
| AgentAdmin | Agent管理员 | 管理其他Agent角色 |
| SupplyModifier | 供应修改者 | mint/burn代币 |
| Freezer | 冻结者 | 冻结/解冻地址和代币 |
| TransferManager | 转账管理者 | 执行强制转账 |
| RecoveryAgent | 恢复代理 | 恢复丢失的钱包 |
| ComplianceAgent | 合规代理 | 管理合规模块 |
| WhiteListManager | 白名单管理者 | 管理身份注册表 |

Owner角色体系专注于系统级配置和关键合约管理，包含7个子角色：

| 角色名称 | 描述 | 权限范围 |
|:-------:|:----:|:-------:|
| OwnerAdmin | Owner管理员 | 管理其他Owner角色 |
| RegistryAddressSetter | 注册表设置者 | 设置IR/TIR/CTR地址 |
| ComplianceSetter | 合规设置者 | 设置合规合约 |
| ComplianceManager | 合规管理者 | 调用合规函数 |
| ClaimRegistryManager | Claim注册管理 | 管理claim topics |
| IssuersRegistryManager | 发行者注册管理 | 管理trusted issuers |
| TokenInfoManager | Token信息管理 | 修改name/symbol/onchainID |

总体的权限图：

```bash
┌─────────────────────────────────────────────────────────────┐
│                      Owner (最高权限)                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       │                               │
┌──────▼───────┐              ┌────────▼────────┐
│ OwnerManager │              │  AgentManager   │
│ (系统配置)    │              │   (日常运营)      │
└──────┬───────┘              └──────────┬───────┘
       │                                 │
       │ 委托                             │ 委托
       │                                 │
┌──────▼──────────────────┐      ┌───────▼───────────────┐
│ 7个Owner子角色           │      │ 7个Agent子角色          │
│ - OwnerAdmin            │      │ - AgentAdmin          │
│ - RegistryAddressSetter │      │ - SupplyModifier      │
│ - ComplianceSetter      │      │ - Freezer             │
│ - ComplianceManager     │      │ - TransferManager     │
│ - ClaimRegistryManager  │      │ - RecoveryAgent       │
│ - IssuersRegistryManager│      │ - ComplianceAgent     │
│ - TokenInfoManager      │      │ - WhiteListManager    │
└─────────────────────────┘      └───────────────────────┘
```

**可升级代理系统（Proxy）**

T-REX采用代理模式实现合约可升级性，允许在不更改合约地址的情况下升级业务逻辑。

**工厂系统（Factory）**

该部分实现了T-REX套件的自动化部署系统，使用CREATE2操作码确保跨链地址一致性。工厂模式极大简化了复杂的部署流程，将原本需要多个交易的部署压缩为单个交易。

### 完整流程

**1.部署身份注册系统**

```solidity
// 部署ClaimTopicsRegistry（声明主题注册表）
ClaimTopicsRegistry claimTopicsRegistry = new ClaimTopicsRegistry();
// 配置必需的声明主题
claimTopicsRegistry.addClaimTopic(1); // 1 = KYC
claimTopicsRegistry.addClaimTopic(2); // 2 = AML
// 此时系统只认投资者的KYC，AML的声明，必须得拥有才能参与（是都得拥有还是只需要拥有一个，是看Verify的requiredClaimTopics字段）

// 部署TrustedIssuersRegistry（可信发行者注册表）
TrustedIssuersRegistry trustedIssuersRegistry = new TrustedIssuersRegistry();
// 部署一个ClaimIssuer（KYC服务商）
ClaimIssuer claimIssuer = new ClaimIssuer(claimIssuerAddress);
// 将KYC服务商添加为可信发行者，并指定它可以颁发哪些主题的声明
uint256[] memory allowedTopics = new uint256[](2);
allowedTopics[0] = 1; // 允许颁发KYC声明
allowedTopics[1] = 2; // 允许颁发AML声明
trustedIssuersRegistry.addTrustedIssuer(IClaimIssuer(address(claimIssuer)), allowedTopics);
// 此时，由该服务商颁发的KYC和AML的声明可以被系统认可

// 部署IdentityRegistryStorage（身份注册存储）
IdentityRegistryStorage identityStorage = new IdentityRegistryStorage();
// 部署IdentityRegistry（身份注册表）
IdentityRegistry identityRegistry = new IdentityRegistry(
    address(trustedIssuersRegistry),
    address(claimTopicsRegistry),
    address(identityStorage)
);
// 将IdentityRegistry绑定到IdentityRegistryStorage
identityStorage.bindIdentityRegistry(address(identityRegistry));
```

此时系统拥有了完整的身份注册系统identityRegistry，claimIssuer是可信的声明发行商，并且允许颁发KYC和AML；

**2.部署合规系统**

```solidity
// 部署ModularCompliance（模块化合规合约）
ModularCompliance compliance = new ModularCompliance();

// 部署、添加和配置合规模块
// 部署国家白名单模块
CountryAllowModule countryAllowModule = new CountryAllowModule();
compliance.addModule(address(countryAllowModule));
// 配置国家白名单：只允许中国(156)和美国(840)的投资者
bytes memory countryAllowData = abi.encodeWithSignature(
    "batchAllowCountries(uint16[])",
    [156, 840] // 国家代码数组
);
compliance.callModuleFunction(countryAllowData, address(countryAllowModule));

// 部署供应量限制模块
SupplyLimitModule supplyLimitModule = new SupplyLimitModule();
compliance.addModule(address(supplyLimitModule));
// 配置供应量限制：最多1000万个代币
bytes memory supplyLimitData = abi.encodeWithSignature(
    "setSupplyLimit(uint256)",
    10_000_000 * 10**18 // 1000万个代币（假设18位小数）
);
compliance.callModuleFunction(supplyLimitData, address(supplyLimitModule));

// 部署转账费用模块（可选）
TransferFeesModule transferFeesModule = new TransferFeesModule();
compliance.addModule(address(transferFeesModule));
```

此时的系统拥有了合规模块，并且支持3个模块：国家白名单、供应量限制、转账费用模块；

**3.部署代币合约**

```solidity
// 部署Token合约
Token token = new Token(
    address(identityRegistry),    // 身份注册表地址
    address(compliance),           // 合规合约地址
    "Security Token",              // 代币名称
    "SEC",                         // 代币符号
    18,                            // 小数位
    address(0)                     // onchainID（可选，初始可设为0）
);

// 将Token合约绑定到Compliance合约
compliance.bindToken(address(token));

// 为发行方授予必要的权限（使用AgentRole系统）
token.addAgent(tokenIssuer); // tokenIssuer获得Agent权限，可以mint代币
```

**4.为投资者创建身份并颁发声明**

```solidity
// 投资者1创建ONCHAINID
// 投资者1部署自己的Identity合约
Identity investor1Identity = new Identity(investor1, true);
// 投资者1将自己的地址添加为MANAGEMENT key
bytes32 investor1Key = keccak256(abi.encode(investor1));
investor1Identity.addKey(investor1Key, 1, 1); // purpose=1(MANAGEMENT), keyType=1(ECDSA)

// KYC机构为投资者1颁发声明
// KYC机构线下验证投资者1的身份后，在链上颁发KYC声明

// 准备声明数据
bytes memory claimData = abi.encode(investor1, "KYC Verified");

// KYC机构对数据签名
bytes32 dataHash = keccak256(claimData);
bytes memory signature = sign(dataHash, claimIssuerPrivateKey); // 使用KYC机构私钥签名
// 将声明添加到投资者1的Identity合约
// 注意：需要投资者1授权，或者在部署Identity时就给claimIssuer添加CLAIM权限
investor1Identity.addClaim(
    1,                              // topic = 1 (KYC)
    1,                              // scheme = 1 (ECDSA)
    address(claimIssuer),           // issuer
    signature,                      // 签名
    claimData,                      // 数据
    ""                              // URI（可选）
);
```

**5.注册投资者到身份注册表**

```solidity
// 代币发行方将投资者1注册到系统中
identityRegistry.registerIdentity(
    investor1,                        // 用户地址
    IIdentity(address(investor1Identity)), // 身份合约
    156                               // 国家代码：中国
);

// 验证投资者是否通过认证
bool isInvestor1Verified = identityRegistry.isVerified(investor1); // 返回 true
```

`isVerified` 会检查：

1. 投资者是否已注册
2. 投资者的Identity合约中是否有有效的KYC声明
3. 声明是否由可信的发行者颁发
4. 签名是否有效

**6.铸造代币给投资者**

```solidity
// 代币发行方铸造100个代币给投资者1
// 注意：只有具有 Agent 权限的地址才能调用 mint
token.mint(investor1, 100 * 10**18); // 100个代币

// 查询余额
uint256 balance = token.balanceOf(investor1); // 100 * 10**18
```

**7.投资者之间转账**

```solidity
// 投资者1向投资者2转账10个代币
// 作为investor1调用
token.transfer(investor2, 10 * 10**18);

// 转账流程：
// 1. Token.transfer() 被调用
// 2. 检查 identityRegistry.isVerified(investor2) → true（投资者2已验证）
// 3. 检查 compliance.canTransfer(investor1, investor2, 10*10^18)
//    - CountryAllowModule检查
//    - SupplyLimitModule检查
//    - 所有模块都通过 → canTransfer返回true
// 4. 执行ERC20转账逻辑
// 5. 调用 compliance.transferred(investor1, investor2, 10*10^18) 更新模块状态
// 6. 转账成功
```

**8.动态添加新的合规模块**

假设监管机构要求添加"最大持仓限制"；

```solidity
// 部署新的合规模块
MaxBalanceModule maxBalanceModule = new MaxBalanceModule();

// 添加到合规合约
compliance.addModule(address(maxBalanceModule));

// 配置模块：每个投资者最多持有1000个代币
bytes memory maxBalanceData = abi.encodeWithSignature(
    "setMaxBalance(uint256)",
    1000 * 10**18
);
compliance.callModuleFunction(maxBalanceData, address(maxBalanceModule));
```

此后，所有转账和铸造都会检查这个新规则，如果某个投资者尝试接收代币导致余额超过1000，交易会失败；

#### 完整流程总结图

```bash
部署阶段：
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Claim   │────▶│ Trusted │────▶│Identity │────▶│Identity │
│ Topics  │     │ Issuers │     │ Storage │     │Registry │
│Registry │     │Registry │     │         │     │  (聚合) │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
                                                      │
                                                      │ 被Token引用
                                                      ▼
┌─────────┐     ┌─────────┐                    ┌─────────┐
│Compliance│◀───│ Module  │◀───────────────────│  Token  │
│         │     │ 1,2,3...│                    │ Contract│
└─────────┘     └─────────┘                    └─────────┘

配置阶段：
1. 添加必需的 Claim Topics (KYC, AML...)
2. 添加可信的 Issuers
3. 部署并配置合规模块
4. 绑定 Token ↔ Compliance

用户注册阶段：
投资者 ──┐
        │ 1. 部署Identity合约
        ▼
    Identity ──┐
              │ 2. KYC机构颁发Claim
              ▼
         Identity (含Claim) ──┐
                              │ 3. 注册到IdentityRegistry
                              ▼
                       IdentityRegistry
                              │
                              │ 4. 验证通过
                              ▼
                           可以接收代币

代币使用阶段：
发行方 ──mint()──▶ Token ──▶ 投资者1
                     │
投资者1 ──transfer()──┤
                     │
                     ├─▶ 检查身份验证 (IdentityRegistry)
                     │
                     ├─▶ 检查合规规则 (Compliance + Modules)
                     │
                     └─▶ 如果全部通过 ──▶ 转账成功 ──▶ 投资者2
```

### 总结

T-REX协议通过三层架构实现了链上证券型代币的完整生命周期管理：

1. **身份层（Identity Layer）**：基于ONCHAINID，为每个参与者创建链上身份容器，存储和管理KYC/AML等合规声明
2. **合规层（Compliance Layer）**：采用模块化设计，支持动态添加各类合规规则，如国家限制、供应量控制、转账费用等
3. **代币层（Token Layer）**：在标准ERC-20基础上扩展了冻结、强制转账、钱包恢复等企业级功能

整个系统的核心优势在于：

- **可扩展性**：模块化架构允许灵活添加新的合规规则，无需修改核心合约
- **合规性**：每笔交易都经过身份验证和合规检查，确保符合监管要求
- **可升级性**：采用代理模式，支持逻辑合约升级而不改变合约地址
- **权限管理**：细粒度的角色权限系统，清晰划分Owner和Agent职责


