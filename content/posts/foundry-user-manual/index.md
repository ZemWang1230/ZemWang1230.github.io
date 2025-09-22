---
date: '2025-05-30T19:58:33+08:00'
title: 'Foundry User Manual'
author: ["Zem"]
summary: "Some usage of Foundry."
tags: ["Solidity", "Test", "Audit"]
series: ["User Manual"]
---
> 自用，随时添加

创建地址:

```solidity
address user = makeAddr("user");
```

设置余额:

```solidity
// ETH
vm.deal(address(user), 1 ether);
// ERC20
deal(address(ERC20token), address(user), 1 ether);
```

发送余额：

```solidity
// ETH
payable(userB).transfer(1 ether);
```

查看余额：

```solidity
// ETH
address(user).balance;
// ERC20
token.balanceOf(address(user));
```

设置谁来调用:

```solidity
vm.startPrank(address(user));
... // 中间所有操作均为user
vm.stopPrank();

vm.startPrank(address(user), address(origin_user));
... // 中间所有操作为user，tx.origin为user_origin
vm.stopPrank();

vm.prank(address(user)); // 设置下一次的调用为user
vm.prank(address(user), address(origin_user)) // 设置msg.sender=user，tx.origin=origin_user
vm.prank(address(msgSender), bool(delegateCall)) // 若delegateCall为true，则下一次的delegatecall的msg.sender是msgSender
vm.prank(address msgSender, address txOrigin, bool delegateCall)
```

预期的revert:

```solidity
// 需要写在调用函数的上面
vm.expectRevert(); // 无论什么revert

vm.expectRevert("err message"); // 只会承认revert输出字符串是"err message"的

vm.expectRevert(ContractXXX.Error_01.selector); // 只会承认revert是指定的Error_01
```

判断值是否相等:

```solidity
assertEq(token.balanceOf(attacker), 5 ether);

assertNotEq(address_A, address_B);

assertTrue(A == B, "Not Equal"); // 需要A == B
assertFalse(A == B, "Equal"); // 需要A != B
```

获取slot中的值:

```solidity
bytes32 implementationSlot = bytes32(0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc);
bytes32 slotValue = vm.load(address(Proxy), implementationSlot);
address implementationAddress = address(uint160(uint256(slotValue)));
```

打印日志:

```solidity
string memory message = "Zem";
console.log("Message:", message);

string memory message_2 = vm.toString(address(user).balance);
console.log("Message_2:", message_2);
```

签名:

```solidity
// 使用私钥签名，并验证是否为签名者
(address user, uint256 userPrivateKey) = makeAddrAndKey("user");
bytes32 messageHash = keccak256("Signed by user");
(uint8 v, bytes32 r, bytes32 s) = vm.sign(userPrivateKey, messageHash);
address signer = ecrecover(messageHash, v, r, s);
assertEq(user, signer);

// 使用Wallet结构签名
Wallet memory user_wallet = vm.createWallet("user");
bytes32 messageHash = keccak256("Signed by user");
(uint8 v, bytes32 r, bytes32 s) = vm.sign(user_wallet, messageHash);
address signer = ecrecover(messageHash, v, r, s);
assertEq(user_wallet, signer);
```

设置block.timestamp:

```solidity
vm.warp(123456789);
```

设置block.number:

```solidity
vm.roll(100);
```

设置fee:

```solidity
vm.fee(25 gwei);
```

设置链上分叉：

```solidity
// 先在foundry.toml中写上rpc
[rpc_endpoints]
mainnet = "https://eth-mainnet.g.alchemy.com/v2/xxx"
// 从指定区块分叉
vm.createSelectFork("mainnet", 123456);
// 从指定交易开始分叉，会回滚到该交易的区块，并重放之前的交易
vm.createSelectFork("mainnet", bytes32(tx);
