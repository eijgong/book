# 以太坊智能合约安全性攻防实例解析

## 摘要

&emsp;&emsp;本次内容由zhangzidong同学准备并分享，由龚劼同学进行整理复盘。zidong同学通过几个实例分析了目前以太坊智能合约安全性的问题，并归纳总结了一些具体的经验与方案，以提高开发者在编写智能合约时的安全性。

## 以太坊智能合约弱点

### solidity

1. 外部合约调用（e.g.恶意匿名函数）
2. 异常处理的不一致（e.g.直接调用合约函数与使用call来调用）
3. 使用address.send发送Ehter可能会抛出Out-of-gas（OOG）异常（a.k.a gasless send）
4. 类型转换（e.g. 合约地址cast）
5. Reentrancy（e.g. DAO）
6. 在状态变量中存储隐私数据（e.g. 石头剪子布游戏）
7. 整形溢出（e.g. 1wei攻击）
8. 谨慎使用循环语句（e.g. gas可能不足）
9. 不要使用tx.origin进行授权（e.g. A->B->A结合reentry进行攻击）
10. [solidity编译器的bug](http://solidity.readthedocs.io/en/develop/bugs.html#known-bugs)

### EVM

1. 谨慎使用随机数（i.e. EVM字节码的执行是deterministic的）
2. Bug无法修复（i.e. 代码即法律）
3. Stack深度限制
	- <https://github.com/ethereum/EIPs/blob/master/EIPs/eip-150.md>
	- 区块2463000前是1024
	- 之后改为caller最多只能使用所提供gas的63/64（大致等同于限制在约340的stack深度）

### Blockchain

1. 状态不可预知（e.g. state被其他人修改；区块链分叉；之前黑客大赛observer的用途）
2. 时间戳依赖（e.g. 矿工有可能攻击合约）

## DAO attack实例

![](https://i.imgur.com/M9onUro.png)

### 背景描述

&emsp;&emsp;在SimpleDAO合约中，用户可以donate财产进入合约，合约更新credit值，addressA用户可以取走credit[addressA]财产，合约更新credit[addressA]。

### 攻击逻辑

SimpleDAO合约中存在的问题：

1. 调用外部函数；
2. reentry；
3. 未考虑到整形溢出问题；

攻击步骤：

1. Mallory2合约中先向SimpleDAO中打入1wei；
2. 调用SimpleDAO中的withdraw函数；
3. withdraw函数向Mallory2打入财产，并调用了其匿名函数，此时并未执行credit[msg.sender]-=amount；
4. 由于Mallory2合约中performAttack锁为true，再次调用withdraw，并令performAttack为false；
5. SimpleDAO再次向攻击者打入财产，因锁为true，匿名函数调用结束，合约执行两次withdraw函数中的credit[msg.sender]-=amount，更新为极大值；
6. 攻击者调用getJackpot函数取走合约中所有财产。

### 解决方式

![](https://i.imgur.com/SS0zkzw.png)

1. 四则运算使用safeMath防止溢出；
2. SimpleDAO中第9和10行交换位置，先更新数据在转移财产；
3. 或者使用mutex。

### 防止reentry的方式

1. 采用Checks-Effects-Interactions的模式（i.g. 判断条件、数据更新、外部交互）
2. 像传统多线程编程中一样，采用mutex（也有可能出现攻击者不释放锁的情况）

### 防止整形overflow和underflow的方式

1. 使用safeMath库
2. 加条件判断语句

## KotET 实例

![](https://i.imgur.com/RztsmAb.png)

### 背景描述

&emsp;&emsp;该合约是一个抢夺合约王冠的游戏，合约包含三个变量king、claimPrice、owner。用户可以向合约打钱，若少于claimPrice则throw，否则成为该合约的king，而前一个king则获取奖励，claimPrice更新。

### 攻击逻辑
该合约中存在的问题：

1. 在17行未check返回值，调用匿名函数后若执行失败，导致前一个king无法得到应有的收益，owner获益。

攻击手段

1. 外部函数调用（即匿名函数问题）；
2. gasless send（send只有2300gas的额度）；
3. 异常处理的不一致（exception disorder）。

### 改进版

![](https://i.imgur.com/ujbUpKW.png)

&emsp;&emsp;即查看call的返回值，false则抛出异常。该合约依然有缺陷：king可以使自己的匿名函数直接抛出异常，则king将无法被reclaim。

### 终极改进版

![](https://i.imgur.com/8fCGjUh.png)

&emsp;&emsp;即加入withdraw函数，前任king自行取钱。

### 收发Ether的注意事项

![](https://i.imgur.com/BelAqQE.png)

## 石头剪子布游戏 实例

### 问题背景

&emsp;&emsp;呃，就是区块链上的石头剪子布

### 攻击逻辑

&emsp;&emsp;利用公链无隐私的特性，A将自己要出的写入链中，B从链上读取数据，并写入自己必赢的招式。

### 解决方案

- Secure multi-party computation
- Zk-snark

## Governmental 实例

![](https://i.imgur.com/24pJ1gt.png)

### 背景描述

&emsp;&emsp;类似上面的KotET实例，参与者向合约打钱，当最后一个参与者打完钱之后，12小时内没有其他人参与，则该参与者获得合约中绝大多数的Ether，成为最大赢家。

### 攻击手段

&emsp;&emsp;合约中存在两个数组型的状态变量，用来记录参与者的地址以及他们打入合约的Ether数量。当12小时过去后，参与者获得收益，合约清空两个数组。由于EVM清空array的方式是一个一个去清空里面的所有元素，当array足够大时，这个操作就会超过gas的最大值，导致永远卡住。

### 解决方案

&emsp;&emsp;不清空array，而是引入一个状态变量以记录array中有多少值，清空是该变量赋值为0，新插入的值继续从头插入。

![](https://i.imgur.com/MRzoKXL.png)

## Governmental简化版举例

![](https://i.imgur.com/vYfkXTK.png)

### 背景描述

&emsp;&emsp;代码如上图所示，调整了等待间隔，具体定义了jackpot。

###攻击逻辑

&emsp;&emsp;存在问题：在硬分叉之前，stack有1024层的限制。

&emsp;&emsp;攻击手段：

![](https://i.imgur.com/tLZPlnl.png)

- 攻击者attack时进行无用的循环增加stack深度。其他如上图所示。造成的后果是钱依然留在合约中，在下一次游戏时可以被owner取走。

![](https://i.imgur.com/DNCKkqb.png)

- 矿工作恶，如上图所示。

## 动态库 实例

![](https://i.imgur.com/f0vpeCD.png)

### 背景描述

&emsp;&emsp;Bob合约中想要使用version函数，而这个函数由SetProvider管理。

###攻击逻辑

&emsp;&emsp;存在的问题：使用了他人管理的动态库。

&emsp;&emsp;攻击手段：SetProvider的拥有者可以替换version函数，当Bob调用函数时则使用了替换后的函数而不自知。

### 解决方式

1. 尽量不使用动态库；
2. 使用仅自己有权限修改的动态库。

## 总结

![](https://i.imgur.com/kx5Owz2.png)

## 参考文献

- [A survey of attacks on Ethereum smart contracts](http://blockchain.unica.it/projects/ethereum-survey/)
- [Solidity security considerations](http://solidity.readthedocs.io/en/develop/security-considerations.html)
- Smart contract best practices
	- <https://consensys.github.io/smart-contract-best-practices/>
	- <https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-zh.md>