---
title: 初识区块链
comments: true
toc: true
categories:
  - blockchain
tags:
  - blockchain
date: 2018-08-15 10:31:31
---
初识区块链。
<!-- more -->

## 为何关注区块链？

[我是怎么入坑的 朱赟](https://mp.weixin.qq.com/s/X38YbpdMGF5svtghwwBI0Q)
[区块链技术人才现状：供给严重不足](https://time.geekbang.org/column/article/12458)

> 我对区块链的评价是这样的，一场乌托邦实验，从最终来看，所谓革命不过是一厢情愿。
>但是，但是即便是乌托邦实验，时间窗口也可以维持足够长，想想德国人的某某主义都一百多年了还有人信不是吗，所以，至少十年，甚至更长，这个产业的饭碗是有的。 -- CaoZ

- - - - - 

正经的笼统介绍：

1. 区块链实现了了无需依赖可信第三方的点对点价值交换。
2. 分布式，去中心化和开放的账本，账本的数据同时分布在多个节点上。
3. 账本数据只能添加，不能修改。每个交易一旦发生即被持久地保存到所有节点上。
4. 无需可信的第三方来保障交易的有效性，安全性以及最终交易的最终确认。
5. 与其他以太网应用共存。
6. 正如TCP/IP协议被设计为创造开放的互联网环境，区块链被设计为创造去中心化的信任环境。

一个典型的区块链如图：
![](/image/2018-08-15/2018-08-15-22-11-26.jpg)

一个极简的区块链实现：[blockchain_go](https://github.com/Jeiwan/blockchain_go)

## 区块链核心技术

**区块链并非是一种颠覆式技术，而是多种技术的集成式创新**

![](/image/2018-08-15/2018-08-15-22-43-47.jpg)


类比TCP/IP协议的分层架构设计，不同区块链的设计一般也包含以下层次：

![](/image/2018-08-15/2018-08-15-22-26-15.jpg)

- - - - - 

一个中心化的系统：

![](/image/2018-08-15/Beginning Blockchain 2018-08-15 22-23-53.png)

一个去中心化的系统：
![](/image/2018-08-15/2018-08-15-22-24-47.jpg)



### 密码学
区块链主要借助密码学提供的几个特性：
* 保密性（Confidentiality）
* 完整性（Data Integrity）
* 身份认证（Authentication）
* 不可否认性（Non-repudiation）

>想象一个场景，我给文拓寄出了一个顺丰快递，我如何保障这个快递没有被别人获知内容，而且内容没有被掉包？文拓又如何确认这是我寄出的包裹？如果我否认寄出过这个包裹，他如何举证？


#### 对称加密：可靠信道
常用实现：AES

![](/image/2018-08-15/2018-08-15-23-09-03.jpg)

缺点：
* 密钥需要通过可靠信道预先协商
* 发送方和接收方必须都是诚实可信的
* 在一个多节点网络中，实现安全通信需要n(n–1)/2数量的密钥

#### 非对称加密：公钥公开
常用实现：RSA
* 公钥公开，所有人都可以获取，用于内容加密或者验签
* 私钥需要妥善保管，用于内容解密或者创建签名

##### 公钥加密的场景：加密通信

![](/image/2018-08-15/2018-08-15-23-05-12.jpg)
##### 私钥加密的场景：身份认证

![](/image/2018-08-15/2018-08-15-23-05-23.jpg)


在多节点加密通信中需要的密钥数量对比：
![](/image/2018-08-15/2018-08-15-23-15-42.jpg)

#### HASH
![](/image/2018-08-15/2018-08-15-23-17-57.jpg)
常用实现：HMACs，SHA512

* 校验数据完整性
* 建立索引
* 可重放
* 不可逆

#### 应用：计算机通信领域的HTTPS加密通道设计
![](/image/2018-08-15/2018-08-15-23-00-20.jpg)
#### 区块链应用

##### 身份认证
Bitcoin通过独有的一对非对称密钥来表明身份。公钥用于收款，私钥用于花钱。

![](/image/2018-08-15/2018-08-15-23-28-16.jpg)

Bitcoin钱包地址生成策略：

![](/image/2018-08-15/2018-08-15-23-24-05.jpg)

一个典型的以太坊发起交易的代码示例：

```javascript
var tx = {
    from: "0xb31FCEF59493cd3f39a41A311Df42AaDd7a5079e",
    gasPricde: "2000000000",
    gas: "42000",
    // 使用公钥作为目标地址
    to: "0x8D08b0bBE8B3Aab4aE4B00eB15fE53C442f9DE47",
    value: "500000000000000000",
    data: ""
};
// 使用私钥签署交易
web3.eth.accounts.signTransaction(tx, '0x81adb7a5b3df9eb15221610954cb3c8e626556afd0fb466b8a6335aa561bbdbe').then(function(signedTx) {
    web3.eth.sendSignedTransaction(signedTx.rawTransaction).then(console.log);
})
```

##### 数据校验，交易查找
UTXO的HASH作为交易的起点：（transaction input中）
![](/image/2018-08-15/2018-08-15-23-36-42.jpg)

一个Bitcoin上的交易如：[举个栗子](https://blockchain.info/tx/b657e22827039461a9493ede7bdf55b01579254c1630b0bfc9185ec564fc05ab?format=json)

Block的HASH指向上一个Block：（block header中）
![](/image/2018-08-15/2018-08-15-23-45-25.jpg)

Merkle Trees通过HASH确保Block中的每个交易都不可被篡改：
![](/image/2018-08-15/2018-08-15-23-50-19.jpg)


### 分布式共识
之前写过两篇相关的文章：[分布式系统设计迷思](https://suclogger.me/分布式系统设计迷思) 和 [分布式系统设计迷思（二）](https://suclogger.me/分布式系统设计迷思（二）)  ，感兴趣的童鞋可以看看。

Paxos协议可以完美的解决在一个**可信环境**中的共识问题。
然而区块链技术本身就是要实现一个分布式的信任机制，显然采用Paxos是个循环证明。如何达成`共识`的命题在这样一个分布式的，缺乏信任（相对于有可信第三方做中间人）的环境中显得尤为重要，因为随时有敌人或者竞争对手试图篡改你的数据，因而我们每当要将一个区块持久化之前，必须要一个适当的机制来进行校验。**共识是商定确定性交易顺序和过滤无效交易的过程。**



>在古代，一些拜占庭的将军率领他们的部队要攻占敌人的一个城池, 每个将军只能控制他们自己的部队并且通过信使传递消息给其他的将军(这条消息只有参与的两个将军知道，其他的将军可以打听，但是不能验证消息的正确性)。如果这些将军中有些是来自敌方的奸细，那么如何使忠诚的将军仍然可以达成行动协议而不受奸细的挑拨，就是拜占庭将军问题。分布式系统的每一个节点可以类比成将军，服务器之间的消息传递可以类比成信使，服务器可能会发生错误而产生错误的信息传达给其他服务器。因此拜占庭容错系统是指：在一个拥有n台服务器的系统中，整个系统对于每一个请求需满足以下两个条件：
>1. 所有非拜占庭服务器使用相同的输入信息，产生一致的结果；
>2. 如果输入的信息正确，那么所有非拜占庭服务器必须接受这个信息，并计算相应的结果。


在一个拜占庭容错系统下达成共识一般有以下方法：
1. PBFT（the practical byzantine fault tolerance algorithm）
2. PoW（the proof-of-work algorithm）
3. PoS（the proof-of-stake algorithm）
4. DPoS（the delegated proof-of-stake algorithm）

#### PBFT

采用PBFT作为共识机制的区块链实现有：Ripple，Hyperledger，Stellar等。
简而言之，PBFT在每一轮中，每个将军基于自己的判断做出决定，收到别的将军的决定后，与自己的决定做某些比较和计算，两两交互之后可以在一定数量叛徒的情况下得出正确共识（3f+1个节点可以容忍拜占庭服务器不超过f个）

![](/image/2018-08-15/2018-08-16-01-19-04.jpg)

这是几种方法中代价最小的一种方法 ，但是他要求每个参与者对他的表决签名，因而牺牲了参与者的匿名性。

#### PoW
这个恐怕是最有名的共识方法了，也是Bitcoin采用的共识方法。工作量证明的关键特点就是，分布式系统中的请求服务的节点必须解决一个一般难度但是可行（feasible）的问题，但是验证问题答案的过程对于服务提供者来说却非常容易，也就是一个不容易解答但是容易验证的问题。
PoW基于Hash的不可逆特性（想要通过 SHA-256 函数的输出推断出输入在目前来看可能性是可以忽略不计的），允许参与者提交自己的计算结果（Nonce），成功提交正确计算结果（Nonce）的节点会得到奖励，因为他确实付出了相应的计算资源。由于Hash函数的特性，获取Nonce除了遍历别无他法，这个遍历Nonce计算满足难度目标的过程就被成为挖矿。

![](/image/2018-08-15/2018-08-16-01-32-58.jpg)


>为什么可以避免成果被窃取？

由于工作量证明需要消耗大量的算力，同时比特币大约 10min 才会产生一个区块，区块的大小也只有 1MB，仅仅能够包含 3、4000 笔交易，平均下来每秒只能够处理 5~7（个位数）笔交易，所以比特币网络的拥堵状况非常严重，也催生了Off-Chain计算和Lightning Network等概念。

#### PoS
PoS用一个数字签名代替了PoW中的hash函数，这个数字签名代表对应的担保资产。即用已有的币挖矿。
简单来说，就是一个根据你持有货币的量和时间，给你发利息的一个制度，在股权证明POS模式下，有一个名词叫币龄，每个币每天产生1币龄，比如你持有100个币，总共持有了30天，那么，此时你的币龄就为3000，这个时候，如果你发现了一个POS区块，你的币龄就会被清空为0。你每被清空365币龄，你将会从区块中获得0.05个币的利息(假定利息可理解为年利率5%)，那么在这个案例中，利息 = 3000 * 5% / 365 = 0.41个币。
与工作量证明相比，权益证明不需要消耗大量的电力就能够保证区块链网络的安全性，同时也不需要在每个区块中创建新的货币来激励矿工参与当前网络的运行，这也就在一定程度上缩短了达成共识所需要的时间。
由于整个网络只会奖励创建区块的节点，不存在任何惩罚，即便是诚实节点，如果它足够理性，那么它也会在所有它收到的链上同时挖矿。POW里，没人挖的分支很快就会变成孤块被丢弃，但在POS里，如果整个网络足够理性，会出现的情况反而是每条分支都会永远存在因为理性的矿工会同时在所有分支上挖矿。如果只用最长链共识的话，POS本身是没法应对分叉的，必须通过惩罚。总而言之就是通过算法之外的事情解决这个问题，引入激励和惩罚。

更多资料参考：[Proof of Stake FAQs](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQs)

#### DPoS
在DPoS中，每个节点都可以投票选出自己的代理节点参与生成新区块的争夺，得票最多的前 N 个节点会被选择成为区块的创建者，下一个区块的创建者就会从这样一组当选者中随机选取。这就避免了PoS中拥有少数资产的节点没有话语权的问题，而且网络中进行新区块创建争夺的节点越少，网络的处理能力就会越强，性能就会越快。使用委托权益证明的 EOS 能够每秒处理几十万笔交易。缺乏严谨的数学证明。

![](/image/2018-08-15/2018-08-16-02-17-49.jpg)

更多资料参考：[DPOS共识算法 -- 缺失的白皮书](https://steemit.com/dpos/@legendx/dpos)

## 不仅仅是加密货币：智能合约
智能合约：在区块链中，正如所有账本数据是分布式，公开且不可篡改，部署在区块链上的代码片段也有同样的特性，称之为智能合约。区别于传统应用，基于智能合约的DApps无需托管在服务器上，而是以交易Payload的形式存在于区块链网络的所有节点上。

Bitcoin区块中的`ScriptSig`和`ScriptPubKey`是一个初始形态的智能合约，不提供图灵完备的实现。
而以太坊的Ethereum Virtual Machine（EVM）提供的图灵完备的实现使得我们可以在智能合约中做几乎任何事情。
![](/image/2018-08-15/2018-08-16-13-44-27.jpg)

以太坊为驱动区块链链的矿工执行智能合约并防御拒绝服务攻击设计了gas Price机制，而且不同于Bitcoin的一切皆交易，以太坊的账户是有状态的（stateful）。不论是发布合约还是执行合约，都需要付出相应的Ether作为代价。


## 基于以太坊开发一个简单的DApp：选举
![](/image/2018-08-15/2018-08-16-13-53-51.jpg)
### 环境准备

1. 准备node环境

```shell
brew install node
```

2. 安装Truffle

```shell
npm install -g truffle
```

3. 安装Ganache

从[ganache官网](https://truffleframework.com/ganache)下载安装包安装

4. 安装chrome扩展：Meta-Mask


### Coding

#### 1. 初始化示例项目pet-shop

```shell
truffle unbox pet-shop
```

初始化后包含以下目录结构：
`contracts/` 智能合约的文件夹，所有的智能合约文件都放置在这里，里面包含一个重要的合约Migrations.sol
`migrations/` 用来处理部署（迁移）智能合约 ，迁移是一个额外特别的合约用来保存合约的变化。
`test/` 智能合约测试用例文件夹
`truffle.js/` 配置文件

#### 2. 编写智能合约
在`contracts`目录下添加`Election.sol`:

```solidity
pragma solidity ^0.4.2;

contract Election {
    // Model a Candidate
    struct Candidate {
        uint id;
        string name;
        uint voteCount;
    }

    // Store accounts that have voted
    mapping(address => bool) public voters;
    // Store Candidates
    // Fetch Candidate
    mapping(uint => Candidate) public candidates;
    // Store Candidates Count
    uint public candidatesCount;

    function Election () public {
        addCandidate("Candidate 1");
        addCandidate("Candidate 2");
    }

    function addCandidate (string _name) private {
        candidatesCount ++;
        candidates[candidatesCount] = Candidate(candidatesCount, _name, 0);
    }

    function vote (uint _candidateId) public {
        // require that they haven't voted before
        require(!voters[msg.sender]);

        // require a valid candidate
        require(_candidateId > 0 && _candidateId <= candidatesCount);

        // record that voter has voted
        voters[msg.sender] = true;

        // update candidate vote Count
        candidates[_candidateId].voteCount ++;
    }
}
```

#### 3. 编译和准备部署脚本
通过以下命令编译：

```shell
truffle compile
```

在`migrations`目录下添加migrate脚本`2_deploy_contracts.js`：

```javascript
var Election = artifacts.require("./Election.sol");

module.exports = function(deployer) {
  deployer.deploy(Election);
};
```

#### 4. 创建用户接口和智能合约交互

```javascript
App = {
  web3Provider: null,
  contracts: {},
  account: '0x0',
  hasVoted: false,

  init: function() {
    return App.initWeb3();
  },

  initWeb3: function() {
    // TODO: refactor conditional
    if (typeof web3 !== 'undefined') {
      // If a web3 instance is already provided by Meta Mask.
      App.web3Provider = web3.currentProvider;
      web3 = new Web3(web3.currentProvider);
    } else {
      // Specify default instance if no web3 instance provided
      App.web3Provider = new Web3.providers.HttpProvider('http://localhost:7545');
      web3 = new Web3(App.web3Provider);
    }
    return App.initContract();
  },

  initContract: function() {
    $.getJSON("Election.json", function(election) {
      // Instantiate a new truffle contract from the artifact
      App.contracts.Election = TruffleContract(election);
      // Connect provider to interact with contract
      App.contracts.Election.setProvider(App.web3Provider);

      return App.render();
    });
  },

  render: function() {
    var electionInstance;
    var loader = $("#loader");
    var content = $("#content");

    loader.show();
    content.hide();

    // Load account data
    web3.eth.getCoinbase(function(err, account) {
      if (err === null) {
        App.account = account;
        $("#accountAddress").html("Your Account: " + account);
      }
    });

    // Load contract data
    App.contracts.Election.deployed().then(function(instance) {
      electionInstance = instance;
      return electionInstance.candidatesCount();
    }).then(function(candidatesCount) {
      var candidatesResults = $("#candidatesResults");
      candidatesResults.empty();

      var candidatesSelect = $('#candidatesSelect');
      candidatesSelect.empty();

      for (var i = 1; i <= candidatesCount; i++) {
        electionInstance.candidates(i).then(function(candidate) {
          var id = candidate[0];
          var name = candidate[1];
          var voteCount = candidate[2];

          // Render candidate Result
          var candidateTemplate = "<tr><th>" + id + "</th><td>" + name + "</td><td>" + voteCount + "</td></tr>"
          candidatesResults.append(candidateTemplate);

          // Render candidate ballot option
          var candidateOption = "<option value='" + id + "' >" + name + "</ option>"
          candidatesSelect.append(candidateOption);
        });
      }
      return electionInstance.voters(App.account);
    }).then(function(hasVoted) {
      // Do not allow a user to vote
      if(hasVoted) {
        $('form').hide();
      }
      loader.hide();
      content.show();
    }).catch(function(error) {
      console.warn(error);
    });
  },

  castVote: function() {
    var candidateId = $('#candidatesSelect').val();
    App.contracts.Election.deployed().then(function(instance) {
      return instance.vote(candidateId, { from: App.account });
    }).then(function(result) {
      // Wait for votes to update
      $("#content").hide();
      $("#loader").show();
    }).catch(function(err) {
      console.error(err);
    });
  }
};

$(function() {
  $(window).load(function() {
    App.init();
  });
});

```

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!-- The above 3 meta tags *must* come first in the head; any other head content must come *after* these tags -->
    <title>Election Results</title>

    <!-- Bootstrap -->
    <link href="css/bootstrap.min.css" rel="stylesheet">

    <!-- HTML5 shim and Respond.js for IE8 support of HTML5 elements and media queries -->
    <!-- WARNING: Respond.js doesn't work if you view the page via file:// -->
    <!--[if lt IE 9]>
      <script src="https://oss.maxcdn.com/html5shiv/3.7.3/html5shiv.min.js"></script>
      <script src="https://oss.maxcdn.com/respond/1.4.2/respond.min.js"></script>
    <![endif]-->
  </head>
  <body>
    <div class="container" style="width: 650px;">
      <div class="row">
        <div class="col-lg-12">
          <h1 class="text-center">Election Results</h1>
          <hr/>
          <br/>
          <div id="loader">
            <p class="text-center">Loading...</p>
          </div>
          <div id="content" style="display: none;">
            <table class="table">
              <thead>
                <tr>
                  <th scope="col">#</th>
                  <th scope="col">Name</th>
                  <th scope="col">Votes</th>
                </tr>
              </thead>
              <tbody id="candidatesResults">
              </tbody>
            </table>
            <hr/>
            <form onSubmit="App.castVote(); return false;">
              <div class="form-group">
                <label for="candidatesSelect">Select Candidate</label>
                <select class="form-control" id="candidatesSelect">
                </select>
              </div>
              <button type="submit" class="btn btn-primary">Vote</button>
              <hr />
            </form>
            <p id="accountAddress" class="text-center"></p>
          </div>
        </div>
      </div>
    </div>

    <!-- jQuery (necessary for Bootstrap's JavaScript plugins) -->
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js"></script>
    <!-- Include all compiled plugins (below), or include individual files as needed -->
    <script src="js/bootstrap.min.js"></script>
    <script src="js/web3.min.js"></script>
    <script src="js/truffle-contract.js"></script>
    <script src="js/app.js"></script>
  </body>
</html>
```

#### 5. 部署合约，运行用户界面

通过truffle提供的migrate 命令部署合约：

```shell
truffle migrate
```

运行node server：

```shell
npm run dev 
```

[election](https://git.dawanju.net/pufang/election)

## 推荐阅读

[Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook)
[Bitcoin 白皮书](https://bitcoin.org/bitcoin.pdf)
[腾讯区块链方案白皮书](https://trustsql.qq.com/chain_oss/TrustSQL_WhitePaper.html)
区块链核心技术与应用

## 其他

>原则没变啊，不支持发币，不给发币的站台，坚决不碰ico。 --CaoZ

[When do you need blockchain](https://medium.com/@sbmeunier/when-do-you-need-blockchain-decision-models-a5c40e7c9ba1)
![](/image/2018-08-15/2018-08-16-14-03-52.jpg)


