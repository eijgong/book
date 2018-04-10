# POS，从Peercoin撸到TenderMint

## 摘要

&emsp;&emsp;本次内容由李明老师准备并分享，由龚劼同学整理复盘。共识机制一直都是区块链系统的核心，现有区块链系统中常用的共识机制有：POW、POS、DPOS、POOL、PBFT等几类，本次分享主要是介绍POS共识机制。本文以PeerCoin和TenderMint为例详细介绍了POS的两种实现方法，也介绍了PoS共识算法的一些问题。

## POS发展历史

&emsp;&emsp;最早共识理论提出是在上个世纪70年代左右，当时共识理论的实践主要是在飞机系统上，如何避免在组件出错的情况下，飞机依然不出故障。

&emsp;&emsp;1982有几个学者提出了拜占庭将军问题，用将军打仗的比喻，来解释这个共识的问题。[《The Byzantine Generals Problem》](https://people.eecs.berkeley.edu/~luca/cs174/byzantine.pdf)

&emsp;&emsp;1999年有一篇文章[《Practical Byzantine Fault Tolerance》](http://pmg.csail.mit.edu/papers/osdi99.pdf)讲了实用的拜占庭容错的方法。但是此时，PBFT的应用也仅仅在局部高速的系统上有所实践，没有太多的应用场景。

&emsp;&emsp;直到2008年比特币的诞生，这种点对点的货币将共识算法带到了新的领域和环境，提出的Blockchain和POW为共识算法带来了新的技术方案和启示。

&emsp;&emsp;在2011年，Bitcointalk上有个帖子提出，当比特币应用更加广泛时，可能需要一个从工作量证明到权益证明的转换，即节点不再用算力进行投票，而是用比特币的数量（私钥）来进行投票。当时该贴中提出POS的几个好处：1.不需要矿机和硬件支持，手续费会变得很低；2.更加快速的交易确认；3.投票更加可信（因为是和币的持有者直接相关）。

&emsp;&emsp;同年，出现了一个POS的真正实现（上面的帖子只是想法）,sunny king于2011年8月发布了一篇文章[《PPCoin: Peer-to-Peer Crypto-Currency with Proof-of-Stake》](https://pdfs.semanticscholar.org/0db3/8d32069f3341d34c35085dc009a85ba13c13.pdf) 论述了POS的可行性。并于一年后的八月，实现了该产品PeerCoin，并挖出了第一个区块。该项目的出现为后续一些区块链项目产品的设计提供了一些启示。后续的一些币种也是延用了一些PeerCoin的设计，并做了一些改进，推动了POS这个共识算法的发展。


&emsp;&emsp;随着比特币的流行，以及ASIC矿机的出现导致算力越来越集中化，POS的呼声也越来越高，大家都认为POS将成为未来的方向。但是研究发现，经典的POS算法埋藏着相当大的隐患：1.Nothing at stake；2.Long Range Attack（后面会详细讲）。所以从2014年开始，一些先锋的研究者如Vitalik Buterin（Casper the Friendly Finality Gadget领导者）、Vlad Zamfir（Casper the Friendly Ghost领导者）、Jae Kwon（Tendermint领导者）也开始用新的思路来研究这个问题。其中Jae Kwon领导的tendermint项目，这个产品和算法，影响了后续很多链产品，比如以太坊的Casper，它可以说是现代POS的基础。

&emsp;&emsp;PEERCOIN和tendermint在POS的历史上具有非常大的作用，前者是POS的第一个实现，后者是现代POS的第一个实现，并且对后续很多POS的项目有很大的影响。


## POW共识的简单介绍


![](https://i.imgur.com/1eV3Yz2.jpg)

- **图解**

	&emsp;&emsp;block，通过hash来连接，下一个block可以找到上一个；transaction，由merklehash写在header里；矿机需要算一个nounce，nounce则是将这个块header和一个随机数算一个hash，如果满足前N位为零，则表示他是一个合法的块。当出块成功后，挖矿者通过coinbase transaction，根据共识协议的规定给自己打N个币作为奖励。

- **好处**

	&emsp;&emsp;在系统运行的过程中，本质上是各个节点是通过算力来进行投票的。这样做的好处有：1.将坏人挡在门外，不是所有人都能生产block，节点需要通过付出硬件、算力作为挖矿的成本；2.让好人达成一致的判断，即当坏人无法破坏时，让网络中的好人可以通过投票来达成某种共识。

- **问题**

	&emsp;&emsp;POW虽然让系统达成了共识，但是也带来了如下的问题：1. 耗电，现在比特币挖矿的消耗相当于整个丹麦国家的用电量；2. 算力集中，曾经是CPU挖矿，但是随着ASIC出现，使得算力越来越集中在专业的挖矿团队手里。

&emsp;&emsp;根据比特币系统带来的启发，以及POW潜在的问题，就出现了POS的想法。为什么必须用算力挖矿，而不能用其他形式。共识算法的本质是为了把坏人挡在门外，让与系统有经济学关系的人来挖矿，为什么不能让有币的人来生产区块、验证区块、为系统达成共识做出贡献呢。

&emsp;&emsp;这样的想法导致了peercoin的产生，通过借鉴比特币的系统，修改共识算法，用stake来代替hash power


## PeerCoin中POS的实现

&emsp;&emsp;Peercoin是一种Chain-base-style POS(即所有Node都有出块的权利和可能性，因此形成竞争和分叉，主链的概念是通过某种共识达成，如最长链，coinage累积最多的链，消耗能量最多的链等等)。它在早期借鉴了很多比特币的代码，只是把POW的共识算法换成了POS，所以系统在某些方面还是很类似的。

### PROOF

- **POW**

	&emsp;&emsp;POW是通过hash来算Nounce以证明自己付出了成本。如果其他节点在这个块中看到了一个合法的nounce，就证明挖矿者确实花了很多算力在生产区块的过程中。

- **POS**

	&emsp;&emsp;POS中的proof是这样来做的，block的生产者，自己给自己转币，这个过程叫coin stake transaction，相当于nounce的作用。nounce证明了你拥有算力；coin stake证明了你有币，是系统利益的相关者。



![](https://i.imgur.com/Mruj4Ts.jpg)

### 图解

&emsp;&emsp;灰色隐去的部分是pow的做法，黑色部分就是Peercoin的做法，引入了随机性，给最长链的定义提供了参数（coin·age = coin数 * 从上一次到现在未花费的时间），coinage决定了你有多大概率来挖到币。target就类似于比特币的难度调整。左边的proofhash是通过kernel算出来的。这个公式模拟了pow里更大算力就拥有更大的概率挖到矿。一旦创建出coin stake transaction ，coinage就归零，然后重新开始累积。因为很多producer都可以出块，那么定义消耗coinage最多的链就是最长链。

### 过程小结

1. 添加coinbase的transaction（即给自己的奖励）
2. 添加staking transaction（即持币的证明）
3. 准备区块、计算merkle根，计算header
4. 计算kernel（前序六十四个区块的hash，加上当前time stamp，加上用来做staking transaction的coin的前序状态的hash）
5. 计算staking difficulty, difficulty = (base difficulty)/coinage, base difficult 通过协议调整，coinage是producer选择的coin的信息
6. 比较hash(kernal)和staking difficulty，如果kernal > staking difficulty 则满足条件，该区块可以进行广播，但是否能被接受，就要看其他producer的情况；如果小，则需要重新准备区块，可以更换coin，因为kernel的hash与coin前序状态的hash有关
7. 签名区块

### 缺陷

&emsp;&emsp;作为一个最早的POS系统，它存在着一些缺陷。

1. coinage的积攒

	&emsp;&emsp;虽然某个节点所持有的coin不多，但是可以积攒地非常大，当十分大的时候，可以进行一些双花的攻击。比如在一条链上生产区块，花出了币之后，在老的区块上重新进行生产，因为coinage的信息在不同链上都是存在的。

2. 懒节点

	&emsp;&emsp;因为生产区块的概率是通过时间线性增加的，所以无需一直在线，如果本次不参与生产区块，coinage会叠加，则在后续的生产中，可以将期望收益收回（即节点是否在线，对其收益的期望值影响很小）。这种情况会造成节点不活跃，系统安全受到威胁。

&emsp;&emsp;这些问题并非严重，后续一些项目通过将coinage改为coin解决了该问题。后期的项目为吹捧自己的pos，都将peercoin的coinage当靶子以标榜自己。

### 严重问题：

- **nothing at stake**

	&emsp;&emsp;产生这种情况的根本原因是：**生产区块的外界成本很低（算力、硬件资源等）**。当一个node看到分叉时，最理性的选择是在两条链上同时生产区块。如图下所示

![](https://i.imgur.com/xDB4H31.jpg)

&emsp;&emsp;图解：分叉A的概率0.9，分叉B的概率0.1；若什么都不做，则拿不到任何好处。若在A上生产区块，则EV=0.9，在B上生产则是0.1（最终结果要乘以自己能够生产区块的概率）；然而最好的选择是不浪费任何机会成本，因为外界成本趋近为0，所以在两个链上同时生产，EV = 1；

&emsp;&emsp;导致问题：对于经典POS中，对于任何一个理性且利己的挖矿者，会在所有可以挖矿的链上进行挖矿。这样做的好处是，1.增加收益；2.潜在地寻找一些双花的机会。（不同分叉的coinage的程度相差很小，所以一部分很小的coinage可以在不同的链上摇摆选择，产生双花的攻击）

&emsp;&emsp;POW没有这样的问题，因为需要消耗算力，若在A链上消耗，则B链上就少了（外界成本高昂）。这种情况下，矿机挖矿的最优解是挖自己认为收益最大的链的区块。

- **long range attack**

&emsp;&emsp;对于Bitcoin来说，51%攻击的问题是存在的，但是要在离当前区块不远的位置，完成超车的概率才比较大。在POS中，攻击者可以在很古老的区块上进行分叉（比如创世区块）。由于在最古老的状态下，某些节点可能具有非常多的币，而币又可以生币，所以将会导致这些节点占有币的比例越来越高，越来越快地生产区块来做出最长的链，最终会打败原有的链。

在后续的一些POS项目中，针对以上的两种情况进行了一些改良，比如引入惩罚机制解决了nothing at stake 问题；引入弱主观的概念，某种程度上解决了long range attack问题。

## TenderMint

### 背景

&emsp;&emsp;该项目由Jae Kwon领导，在2014年发表了一篇白皮书，思想是基于1999的PBFT，他们能够达到1/3拜占庭容错的效率（即系统里的坏人少于1/3，则系统能达成安全可靠的共识)。最早是一个POS的做法，内置一种币，这个币可以用来抵押来获取产生区块的权利。之后不断演化变为一个consensys engine，通过一个ABCI的消息机制可以扩展，用户可以通过这个engine和library进行扩展，开发自己的应用，比如他们自己的Cosmos Network，比如Ethermint。

### BFT-based PoS

&emsp;&emsp;BFT BASE: 区块的确认在block的粒度上而非chain的粒度。即一个区块的确认不在于他后续区块是怎样的状态。producer在房间里开会投票，直到把一个block投出来，然后向前挪一步，去选择下一个区块。在tendermint里，有一些validator，他们是通过bond transaction使自己的一些币进入stake状态。这些validator有权利去进行proposal来选择区块。投票之后，他们可以unbond transaction将币提出来，退出validator的角色。在一轮内部，validator可以通过vote，签名一个消息来广播达成共识，通过多轮的投票来决定一个block是不是OK的。在vote的时候，一个block不断会进入下一个状态，如果有2/3的多数认为它可以，它则可以进入下一轮。

### Bond的好处

&emsp;&emsp;因为以往POS里没有惩罚的概念，如果一个producer、node在不同链上的工作，链之间彼此互相不知道，所以大家无法知晓是否一个人是在多个区块链上工作，因此产生了nothing at stake 问题。但是一旦有bond，一个producer必须把coin stake起来，这样提供了惩罚的可能，如果一个validator被发现在投了不同的区块时，则他的钱会被没收。如何发现：通过evidence transaction使得制造混乱的validator获得惩罚。这样既解决了nothing at stake 问题，validator如果做了错误的事情，币会被没收，且该bond会在链上存在一段时间。在这个时间窗口内，该validator无法做坏的事情了。

### 产生的问题

&emsp;&emsp;问题：没有最长链的概念，完全根据当前的状态腿短下一个状态是否可靠。举例：李老师在航海，在船上，以前可以飞到空中看到哪个船更大；现在没有最大的船的概念，即我在哪条船，哪条就是正确的船。这里long range attack装换成另一个问题：当一个节点第一次加入网络，无法知道谁是对的；当一个节点offline很长时间（超过了bond的窗口期），那么他对于当前正确的链也有更多的选择。

### 解决的办法

&emsp;&emsp;实际上没有程序和算法的解，不想chain base一样有客观的最长链（通过共识算法、参数就可以看到谁是主链），但是POS就没有这样的概念。妥协：weak subjective，通过媒体、官网等社区认为可信的第三方参与。但是他和经典POS的频繁check ，当节点第一次进入网络，或发生异常长期掉线的情况下进入网络，需要这个行为来帮助他认清主链，但只要他一直跟着主链跑，就不会产生这样的问题

### tenderMint投票流程

![](https://i.imgur.com/YLmggfN.jpg)

1. propose	

	有一个轮转策略，由某一个validator每一轮来propose区块

2. prevote	
 
	广播这个区块，当其他validator收到这个区块后，会选择要不要接受他；prevote结果会广播给其他节点，其他节点发现若该区块被2/3的validator节点的validPower给prevote之后，将会进行一个commit操作。validPower是指你bond transaction中coin的数量，当其越多时，则该power越大。

3. 进行第二轮2/3多数投票	

	如果该区块又收到超过2/3的投票，该validator则进行commit并广播。然后该区块就成立了。大家可以开始去生产下一个区块。

&emsp;&emsp;有一个网络延时的问题，比如掉线或者网速慢，不能及时地产生vote。因为系统认定是超过2/3才能进入下一轮，所以超时情况下，说明他的选择是无效的。

### tenderMint中区块的结构

![](https://i.imgur.com/mvUvlY1.jpg)

&emsp;&emsp;与比特币等区块结构的不同：block通过hash会连接，但是该hash是由header hash、validation hash、transactions hash组成一个merkle树，然后以这个根作为该block的hash。

&emsp;&emsp;为何有这样一个相对复杂的设计：通过投票的过程可以看到，propose的时候已经有一个block，他的hash已经算出来。但最后还要进行validate的操作，因为vote要把自己IDEsignature放进去，还有一些transactions是其他的行为，比如发现一个validator作恶，需要惩罚他，然后发送一个transaction来取走他bond的coin。这样一轮轮投票的机制，先是application data产生的header的hash以及validation data，额外的transaction data他们产生的hash，这样最后这三个data产生的hash再合并后产生一个hash，最后一个hash作为block之间的连接，形成blockchain。

## 总结

&emsp;&emsp;POS是一个非常复杂的问题，他的安全性、性能，整个社区都在研究过程中。今天没有讨论，贿赂如何影响安全、抗监管、validator如何组织某些transaction进入block被确认。还有一些方案没在讨论范围内，今天讨论的是Chain-based，BFT-based的PoS。PoS进入了一个全新的阶段，如以太坊的Casper以及Tendermint都是一种全新的思路，但是还有很多可以讨论的空间。

## 参考


![](https://i.imgur.com/56O9QzO.jpg)