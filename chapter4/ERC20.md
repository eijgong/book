# ERC20发币实践

## 摘要

&emsp;&emsp;本次分享是由田线宝与张岩进行准备和分享，由龚劼同学整理复盘。本次分享是一个实践型工程，因此本文无理论方面的叙述，主要描述了在以太坊测试网络上的发币过程，并配以关键性的图片，发币代码由以太坊官网获取。

## ERC20介绍

&emsp;&emsp;ERC20定义了以太币Token智能合约的接口，每个人都可以根据自己的方法来完成不同的实现，如增发与否，限制总数等等。

## 具体实践

- metamask安装，用基于chrome内核的浏览器以下载；<https://metamask.io/>
- 建立自己的钱包账户；进入后使用BETA版本（功能更多）；选择进入Rinkeby Test Net；

![](https://i.imgur.com/AsptJFJ.png)

- 从bit.do/erc20获取以太坊官网给出的样板代码；
- 复制代码到remix.ethereum.org；
- 点击compile，修改run中的environment为injected Web3；
- 根据数据格式设置参数并点击Create进行部署合约；

![](https://i.imgur.com/AYkCbyI.png)

- 弹出对话框，点击confirm；
- 点击metamask左部菜单栏后Add Token；

![](https://i.imgur.com/aQT2qdh.png)

- 输入已部署合约的地址，并点击确认。（合约地址获取点击下图黄色图标）

![](https://i.imgur.com/NFt3tF6.png)

- 然后就可以自己根据合约提供的功能捣鼓捣鼓，比如说转账什么的咯。

PS：若部署合约时缺少ETH，可在<https://faucet.rinkeby.io/>按要求进行领取：

1. 复制自己的地址到社交软件上上传，如twitter、facebook；
2. 复制发布信息的url到faucet.rinkeby.io中，即可领取。