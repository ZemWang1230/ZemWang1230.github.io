---
date: '2025-06-03T19:00:00+08:00'
title: '使用Foundry和Viem与链上合约交互'
author: ["Zem"]
summary: "Quick start guide to viem."
tags: ["Viem", "Solidity", "Foundry", "TypeScript"]
series: ["学习笔记"]
---

## 合约部分

Foundry新建项目:

```bash
forge init viem-test
```

编译及测试合约:

```bash
forge build
foege test
```

启动本地测试链:

```bash
anvil --chain-id 1234 -b 10
# 链id 1234
# 每10s生成一个区块
```

在` script ` 目录中新建` Deploy.sol `:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "../src/Counter.sol";

contract Deploy is Script {
    function run() external {
        vm.broadcast();
        new Counter();
    }
}
```

部署合约:

```bash
forge script script/Deploy.sol --rpc-url http://127.0.0.1:8545 --broadcast --sender 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

## Viem交互部分

新建一个文件夹，在其中初始化项目，并安装依赖:

```bash
pnpm init

pnpm install viem dotenv
pnpm install -D typescript ts-node @types/node
```

初始化typescript配置文件:

```bash
npx tsc --init
```

创建` index.ts `文件:

```typescript
import { defineChain, createPublicClient, http, createWalletClient } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import dotenv from "dotenv";
import { abi, contract_address } from "./contract/Conunter";
import { availableMemory } from "process";

// 定义本地链，viem集成了很多链，在viem/chains中可以找到
const localChain = defineChain({
    id:1234,
    name: 'Local Testnet',
    network: 'local',
    nativeCurrency: {
        name: 'ETH',
        symbol: 'ETH',
        decimals: 18,
    },
    rpcUrls: {
        default: {
            http: ['http://127.0.0.1:8545'],
        },
    },
    testnet: true
});

// 创建一个公共客户端
const client = createPublicClient({
    chain: localChain,
    transport: http(),
});

// 提取私钥
dotenv.config();
// 生成钱包账户
const account = privateKeyToAccount(`0x${process.env.PRIVATE_KEY}`);
// 创建一个钱包客户端
const walletClient = createWalletClient({
    account,
    chain: localChain,
    transport: http(),
});

// 获取合约中的number
async function getNumber() {
    const number = await client.readContract({
        address: contract_address,
        abi: abi,
        functionName: "number",
    });
    console.log(number);
}

async function main() {
    getNumber();

    // 调用合约中的setNumber，使用writeContract
    const transaction = await walletClient.writeContract({
        address: contract_address,
        abi: abi,
        functionName: "setNumber",
        args: [BigInt(100)],
    });
    // 等待交易被确认，并返回交易收据
    const receipt = await client.waitForTransactionReceipt({ hash: transaction });
    console.log(receipt);

    getNumber();
}

main();
```

创建` contract/Counter.ts `，往其中写入abi及部署的地址，在定义的后面加入` as const `，这样框架更好识别:

```typescript
export const abi = [
    {
        "inputs": [],
        "stateMutability": "nonpayable",
        "type": "function",
        "name": "increment"
    },
    {
        "inputs": [],
        "stateMutability": "view",
        "type": "function",
        "name": "number",
        "outputs": [
            {
                "internalType": "uint256",
                "name": "",
                "type": "uint256"
            }
        ]
    },
    {
        "inputs": [
            {
                "internalType": "uint256",
                "name": "newNumber",
                "type": "uint256"
            }
        ],
        "stateMutability": "nonpayable",
        "type": "function",
        "name": "setNumber"
    }
] as const;

export const contract_address = "0x5FbDB2315678afecb367f032d93F642f64180aa3" as const;
```

创建` .env `文件，存入我们的私钥，供后续发起交易:

```
PRIVATE_KEY=ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

在` package.json `中加入` start `命令:

```json
{
  "name": "viem-interact",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "ts-node index.ts"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "packageManager": "pnpm@10.11.1",
  "dependencies": {
    "dotenv": "^16.5.0",
    "viem": "^2.30.6"
  },
  "devDependencies": {
    "@types/node": "^22.15.29",
    "ts-node": "^10.9.2",
    "typescript": "^5.8.3"
  }
}

```

执行` pnpm start `，可以看到以下结果:

```bash
(common) ➜  viem-interact pnpm start

> viem-interact@1.0.0 start /Users/xxx/Desktop/viem-start/viem-interact
> ts-node index.ts

0n
{
  type: 'eip1559',
  status: 'success',
  cumulativeGasUsed: 43696n,
  logs: [],
  logsBloom: '0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000',
  transactionHash: '0x9c1faf0ec7d1870571b4146528977bdaee0fe33d66fcbaea84b150634c1fd95e',
  transactionIndex: 0,
  blockHash: '0x1f7cd7f75936fc125091fe3ade3cd04ac4e543b5652cd25a4aa7594513d4dcf3',
  blockNumber: 95n,
  gasUsed: 43696n,
  effectiveGasPrice: 1000003547n,
  blobGasPrice: 1n,
  from: '0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266',
  to: '0x5fbdb2315678afecb367f032d93f642f64180aa3',
  contractAddress: null
}
100n
```

