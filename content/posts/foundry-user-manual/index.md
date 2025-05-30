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

设置谁来调用:

```solidity
vm.startPrank(address(user));
... // 中间所有操作均为user
vm.stopPrank();

vm.startPrank(address(user), address(origin_user));
... // 中间所有操作为user，tx.origin为user_origin
vm.stopPrank();

vm.prank(address(user)); // 设置下一次的调用为user
vm.prank(address(user), address(origin_user))
```

预期的revert:

```solidity
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

