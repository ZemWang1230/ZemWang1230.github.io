---
date: '2025-10-24T17:03:06+08:00'
title: 'ERC-4626'
author: ["Zem"]
summary: "Tokenized Vaults with a single underlying EIP-20 token."
tags: ["ERC-20", "ERC-4626"]
series: ["EIP学习笔记"]
---

### ERC4626概要

> ERC4626是一个代币化保险库标准，使用ERC20代币来表示某种其他资产的股份。

` ERC4626合约 `就像一个Vault，将一个` ERC20 `代币` A `存入，然后会获取另一种` ERC20 `代币` B `，此时你所拥有的` B `的数量，就代表你在Vault中所持有的所有` A `的股份；

如果` A `的增长速度大于` B `的增长速度，你就能按股份获得你所存入的` A `的利息；

随后可以将` B `放回Vault，获得原先存入的` A `加上应得的一些` A `的利息；

### ERC4626的动机

> 有一个流动性池子，存入并定期赚取稳定币USDC，此时，USDC就是资产；
>
> 现在想的是，如何来给用户发放收益？

#### 低效的方式

按比例将USDC分发给每个有股份的人，但是这样，首先得在合约中更新每个人的余额，会耗费很多的gas（不止是遍历，修改mapping中的数据也会很耗gas）；

#### ERC4626的方式

每个人存入基础资产，并获得一个股份，最后withdraw或者redeem时，只需要计算该用户的股份占所有股份的多少，然后分配即可，并且期间任何变动（其余用户的存入或取回），合约只需要修改` 股份的总供应量 `和` 存入的资产数量 `，并不需要去修改每个人的份额；

比如我存入10` USDC `，拿到了1个` wUSDC `股份，随后，又有其余用户存入90` USDC `，他会拿到9个` wUSDC `股份，现在合约中已存100` USDC `，一共产生了10个` wUSDC`股份；

现在池子产生收益了，100` USDC `变成了110` USDC `，我想取回` USDC `时，只需要将我所持有的` wUSDC `放回合约，此时我将拿到本金10` USDC `+收益1` USDC `；

### 代码分析

接口：[IERC4626.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/interfaces/IERC4626.sol)

代码：[Openzeppelin的ERC4626.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol)

首先，` ERC4626 `就是` ERC20 `的拓展，构造阶段可以将ERC20资产初始化进合约，并将该ERC20作为基础资产，用户可以存入它来获取利息；

```solidity
constructor(IERC20 asset_) {
    (bool success, uint8 assetDecimals) = _tryGetAssetDecimals(asset_);
    _underlyingDecimals = success ? assetDecimals : 18;
    _asset = asset_;
}
```

所以` ERC4626 `也有` ERC20 `所拥有的` balanceOf `、` transfer `、` transferFrom `、` approve `、` allowance `等关键函数；

合约中存入的是什么资产，以及总共有多少资产（并没有股份的资产地址，因为股份就是ERC4626合约）：

```solidity
/// @inheritdoc IERC4626
function asset() public view virtual returns (address) {
    return address(_asset);
}

/// @inheritdoc IERC4626
function totalAssets() public view virtual returns (uint256) {
    return IERC20(asset()).balanceOf(address(this));
}
```

#### 存入和提取

存入资产` deposit `、` mint `：

```solidity
/**
 * @dev Deposit/mint common workflow.
 */
function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal virtual {
    // 将基础资产转到该合约
    SafeERC20.safeTransferFrom(IERC20(asset()), caller, address(this), assets);
    // 铸造相应数量的股份
    _mint(receiver, shares);
    // 释放事件
    emit Deposit(caller, receiver, assets, shares);
}

/// @inheritdoc IERC4626
// 存入资产，根据所存入的资产数量，计算该mint多少股份
function deposit(uint256 assets, address receiver) public virtual returns (uint256) {
    uint256 maxAssets = maxDeposit(receiver);
    if (assets > maxAssets) {
        revert ERC4626ExceededMaxDeposit(receiver, assets, maxAssets);
    }

    uint256 shares = previewDeposit(assets);
    _deposit(_msgSender(), receiver, assets, shares);

    return shares;
}

/// @inheritdoc IERC4626
// 指定想要多少股份，随后计算该从你那划转多少资产
function mint(uint256 shares, address receiver) public virtual returns (uint256) {
    uint256 maxShares = maxMint(receiver);
    if (shares > maxShares) {
        revert ERC4626ExceededMaxMint(receiver, shares, maxShares);
    }

    uint256 assets = previewMint(shares);
    _deposit(_msgSender(), receiver, assets, shares);

    return assets;
}
```

官方建议只重写` _deposit `，因为` deposit `和` mint `的底层依赖它；

* ` deposit `，**存入一定的资产，然后合约会计算你会得到多少股份**；
* ` mint `，**你指定需要多少股份，然后合约会计算需要从你那划转多少基础资产**；

随后会释放事件` event Deposit(address indexed sender, address indexed owner, uint256 assets, uint256 shares) `，表示sender存了assets数量的资产，然后为owner铸造了shares数量的股份；

取回资产` withdraw `、` redeem `：

```solidity
/**
 * @dev Withdraw/redeem common workflow.
 */
function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares)
    internal
    virtual
{
    if (caller != owner) {
        _spendAllowance(owner, caller, shares);
    }

    _burn(owner, shares);
    SafeERC20.safeTransfer(IERC20(asset()), receiver, assets);

    emit Withdraw(caller, receiver, owner, assets, shares);
}

/// @inheritdoc IERC4626
// 想从合约中提取多少资产，合约会计算销毁多少股份
function withdraw(uint256 assets, address receiver, address owner) public virtual returns (uint256) {
    uint256 maxAssets = maxWithdraw(owner);
    if (assets > maxAssets) {
        revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
    }

    uint256 shares = previewWithdraw(assets);
    _withdraw(_msgSender(), receiver, owner, assets, shares);

    return shares;
}

/// @inheritdoc IERC4626
// 指定要提取的股份数量，合约会计算要提取多少基础资产
function redeem(uint256 shares, address receiver, address owner) public virtual returns (uint256) {
    uint256 maxShares = maxRedeem(owner);
    if (shares > maxShares) {
        revert ERC4626ExceededMaxRedeem(owner, shares, maxShares);
    }

    uint256 assets = previewRedeem(shares);
    _withdraw(_msgSender(), receiver, owner, assets, shares);

    return assets;
}
```

官方同样建议只重写` _withdraw `，因为` withdraw `和` redeem `的底层依赖它；

* ` withdraw `，**想从合约中提取多少基础资产，然后合约会计算销毁你的多少股份**；
* ` redeem `，**指定想提取多少股份，合约会计算该取出你的多少基础资产**；

随后就会释放事件` event Withdraw(address indexed sender, address indexed receiver, address indexed owner, uint256 assets, uint256 shares) `，表示sender销毁了owner的shares数量的股份，让receiver接收了assets数量的基础资产；

#### 预测函数

上面存款和提取的四个主要函数中，都有预测函数：

* ` previewDeposit `，预测存入的资产，会拿到多少股份；
* ` previewMint `，预测要拿到指定数量的股份，需要存多少基础资产；
* ` previewWithdraw `，预测要取回指定数量的基础资产，会销毁多少股份；
* ` previewRedeem `，预测要销毁指定数量的股份，会取回多少资产；

它们都是调用了两个底层函数：

```solidity
/**
 * @dev Internal conversion function (from assets to shares) with support for rounding direction.
 * 将指定数量的基础资产，计算转变为有多少股份
 */
function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
    // shares = assets * (totalSupply + 10**_decimalsOffset) / (totalAssets + 1)
    // totalAssets = 1000 USDC, totalSupply = 1000 wUSDC, 用户存入100 USDC
    // shares = 100 * (1000 + 10**0) / (1000 + 1) = 100 wUSDC
    return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
}

/**
 * @dev Internal conversion function (from shares to assets) with support for rounding direction.
 * 将指定数量的股份，计算转变为有多少资产
 */
function _convertToAssets(uint256 shares, Math.Rounding rounding) internal view virtual returns (uint256) {
    // assets = shares * (totalAssets + 1) / (totalSupply + 10**_decimalsOffset)
    // totalAssets = 1000 USDC, totalSupply = 1000 wUSDC, 用户想存入100 wUSDC
    // assets = 100 * (1000 + 1) / (1000 + 10**0) = 100 USDC
    return shares.mulDiv(totalAssets() + 1, totalSupply() + 10 ** _decimalsOffset(), rounding);
}
```

` 10 ** _decimalsOffset() `的作用：

对齐Vault和Assets两个ERC20代币之间的精度，比如Asset是USDC，decimal为6，而Vault没设置好，设置成18，所以Offset就为18 - 6 = 12，如果不处理，那么在计算 shares 或 assets 时，数值比例会完全错误；

` totalAssets() + 1 `的作用：

早期池子的总额度为0，防止除0；还可以防止通胀攻击；

### 滑点问题

任何代币交换或 vault 操作都有滑点：最终得到的数量 ≠ 预期数量。原因常见有两类：

1. 价格移动或者流动性耗尽：大额交易改变池内余额，从而改变兑换比率，导致成交量少于预期；
2. 被抢跑或者被夹击：交易在链上被别的交易插队，攻击者利用顺序变化获利，受害者拿到更坏的价格；

如果ERC-4626内部的定价不稳健，会导致收到比预期少得多的shares或者assets；

防御的方法之一就是**滑点容错检查**：与Vault交互的合约在调用存款或者取回函数时，带入` minShares `/` minAssets `（用户可接受的最小接收量），合约在操作后对实际收到的数量做检查，若不符合则revert；

### ERC4626的通胀攻击

标准的线性换算公式是：

```solidity
shares = floor( assets * totalSupply / totalAssets )
```

在正常的情况下都是好用的，比如存入1000，totalSupply为1000，totalAssets为1000000；

```solidity
shares = floor(1,000 * 1,000 / 1,000,000) = 1
```

但是若黑客：

1. 在受害者交易前先` donate `，也就是直接向Vault转基础资产，注入资金；
2. 这会把` totalAssets `增加到一个很大的值，而` totalSupply `没有变化（因为donate不会铸造shares）；

当用户按原计划存入1000时，此时的计算结果就会变为：

```solidity
shares = floor(1,000 * 1,000 / 1,000,000,000) = 0
```

所以用户想当如没有存资产进去，攻击者通过不对等的操作改变了share的单位价值（每share现在代表着更多的资产）；

#### 虚拟流动性

所以Openzeppelin中就有了虚拟流动性，也就是` +1 `、` 10**_decimalsOffset() `；

本质上就是在换算公式中把` totalSupply `人为的放大一个常数（通常是` 10**_decimalsOffset() `，既用于精度对齐，也作为虚拟流动性因子）；

所以公式变为了：

```solidity
shares = floor( assets * (totalSupply + 10**_decimalsOffset / (totalAssets + 1) )
```

这样增加了小额入金能获得的shares，使得攻击者需要向池子注入更多的资产（指数级）才能把受害者的shares变为0；

再用上面的例子，用户需要存1000，totalSupply为1000，totalAssets为999999，_decimalsOffset为3，黑客预先存入2000；

```solidity
shares = floor(1,000 * (1,000 + 1,000) / (999,999 + 1) + 2,000)
shares = floor(1,000 * 2,000 / 1,002,000)
shares = floor(2,000,000 / 1,002,000)
shares = 1
```

如果攻击者想把该用户的shares变为0，他必须先存入起码1,000,000数量的资产（也就是用户存入资产的10**3倍），大大提高了攻击的门槛；

### ERC4626金库安全指南

https://learnblockchain.cn/article/17786
