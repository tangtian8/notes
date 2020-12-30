# Fabric2.2.0商业票据业务介绍

## 前提

fabric测试网络已搭建完成

## 商业票据是什么

商业票据，是指由金融公司或某些信用较高的企业开出的无担保短期票据。 商业票据的可靠程度依赖于发行企业的信用程度，可以背书转让，可以贴现。 商业票据的期限在一年以下，利率高于同期银行存款利率，商业票据可以由企业直接发售，也可以由经销商代为发售。 但对出票企业信誉审查十分严格。

## 演示商业票据简介

#### 场景及角色

MagnetoCorp公司发行一张面值 500 万美元的商业票据，DigiBank 认为 MagnetoCorp 公司是值得信赖的，它以 494 万美元的价格购买了MagnetoCorp 公司 6 个月到期的商业票据，DigiBank 完全预计它将能够在 6 个月内从 MagnetoCorp 赎回 500 万美元的商业票据。

| 组织            | 角色                                 |
| --------------- | ------------------------------------ |
| MagnetoCorp公司 | 有发行和确认发行商业票据功能         |
| DigiBank公司    | 对商业票据购买，赎回，查询等交易功能 |

#### 演示的商业票据生命周期

票据 00001 的生命周期相对简单——由于**发行**，**购买**和**兑换**交易，它在 `issued`, `trading` 和 `redeemed` 状态之间转移。

票据 00001 是 5 月 31 号由 MagnetoCorp 发行的。

```javascript
Txn = issue
Issuer = MagnetoCorp
Paper = 00001
Issue time = 31 May 2020 09:00:00 EST
Maturity date = 30 November 2020
Face value = 5M USD
```

该票据被 DigiBank 购买。同一个商业票据如何发生变化：

```javascript
Txn = buy
Issuer = MagnetoCorp
Paper = 00001
Current owner = MagnetoCorp
New owner = DigiBank
Purchase time = 31 May 2020 10:00:00 EST
Price = 4.94M USD
```

6 个月后，如果 DigiBank 仍然持有商业票据，它就可以从 MagnetoCorp 那里兑换：

```javascript
Txn = redeem
Issuer = MagnetoCorp
Paper = 00001
Current owner = HedgeMatic
Redeem time = 30 Nov 2020 12:00:00 EST
```

#### 演示的商业票据的结构

```
Issuer = MagnetoCorp
Paper = 00001
Owner = MagnetoCorp
Issue date = 31 May 2020
Maturity = 30 November 2020
Face value = 5M USD
Current state = issued
```

查看 MagnetoCorp 的票据 `00001` 如何表示为一个状态向量，根据不同的交易刺激进行转换:

![develop.paperstates](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/_images/develop.diagram.6.png)

## 智能合约处理

智能合约定义业务对象的不同状态，并管理对象在不同状态之间变化的过程。智能合约允许架构师和智能合约开发人员定义在区块链网络中协作的不同组织之间共享的关键业务流程和数据。

#### 实现语言

支持两种运行时，Java 虚拟机和 Node.js。支持使用 JavaScript、TypeScript、Java 或其他可以运行在支持的运行时上其中一种语言。

#### 合约类

- [Node.js 智能合约 API 文档](https://hyperledger.github.io/fabric-chaincode-node/)  https://hyperledger.github.io/fabric-chaincode-node/release-2.2/api/

- [Java 智能合约 API 文档](https://hyperledger.github.io/fabric-chaincode-java/)

  JavaScript 类构造函数如何使用其[超类](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)通过一个[命名空间](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/namespace.html)来初始化自身：

  ```
  constructor() {
      super('org.papernet.commercialpaper');
  }
  ```

  在 Java 类中，构造器是空的，合约名会通过 `@Contract()` 注解进行识别。如果不是就会使用类名

`org.papernet.commercialpaper` 非常具有描述性——这份智能合约是所有 PaperNet 组织关于商业票据商定的定义。

#### 交易定义

在类中定位  方法。

Java 标注 `@Transaction` 用于标记该方法为交易定义；TypeScript 中也有等价的标注。

方法中定义的一个额外变量 ctx。它被称为[**交易上下文**](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/transactioncontext.html)，它始终是第一个参数。默认情况下，它维护与[交易逻辑](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/smartcontract.html#交易逻辑)相关的每个合约和每个交易的信息。例如，它将包含 MagnetoCorp 指定的交易标识符，MagnetoCorp 可以发行用户的数字证书，也可以调用账本 API。

#### 交易逻辑

更改状态

#### 对象的表示

- 这是一个内存中的表示;帐本上显示。
- `CommercialPaper` 类扩展了 `State` 类。 `State` 是一个应用程序定义的类，它为状态创建一个公共抽象。所有状态都有一个它们代表的业务对象类、一个复合键，可以被序列化和反序列化，等等。当我们在帐本上存储多个业务对象类型时， `State` 可以帮助我们的代码更清晰。检查 `state.js` [文件](https://github.com/hyperledger/fabric-samples/blob/master/commercial-paper/organization/magnetocorp/contract/ledger-api/state.js)中的 `State` 类。
- 票据在创建时会计算自己的密钥，在访问帐本时将使用此密钥。密钥由 `issuer` 和 `paperNumber` 的组合形成。
- 票据通过交易而不是票据类变更到 `ISSUED` 状态。那是因为智能合约控制票据的状态生命周期。例如，`import` 交易可能会立即创建一组新的 `TRADING` 状态的票据。

#### 访问账本

```
class PaperList extends StateList {
```

此工具类用于管理 Hyperledger Fabric 状态数据库中的所有 PaperNet 商业票据。PaperList 数据结构在[架构主题](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/architecture.html)中有更详细的描述。

帐本中的每个状态数据都需要以下两个基本要素：

- **键（Key）**: `键` 由 `createCompositeKey()` 使用固定名称和 `state` 密钥形成。在构造 `PaperList` 对象时分配了名称，`state.getSplitKey()` 确定每个状态的唯一键。
- **数据（Data）**: `数据` 只是商业票据状态的序列化形式，使用 `State.serialize()` 方法创建。`State` 类使用 JSON 对数据进行序列化和反序列化，并根据需要使用 State 的业务对象类，在我们的例子中为 `CommercialPaper`，在构造 `PaperList` 对象时再次设置。

注意 `StateList` 不存储有关单个状态或状态总列表的任何内容——它将所有这些状态委托给 Fabric 状态数据库。这是一个重要的设计模式 – 它减少了 Hyperledger Fabric 中[账本 MVCC 冲突](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/readwrite.html)的机会。

## 客户端调用处理

在我们的场景中，组织使用应用程序访问 PaperNet，这些应用程序调用定义在商业票据智能合约中的**发行**、**购买**和**兑换**交易。

#### 交易步骤

- 从钱包中选择一个身份
- 连接到网关
- 访问所需的网络
- 构建智能合约的交易请求
- 将交易提交到网络
- 处理响应

#### 钱包

钱包拥有一组身份——X.509 数字证书——可用于访问 PaperNet 或任何其他 Fabric 网络。如果您运行该教程，并查看此目录，您将看到 Isabella 的身份凭证。

想想一个[钱包](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/wallet.html)里面装着政府身份证，驾照或 ATM 卡的数字等价物。其中的 X.509 数字证书将持有者与组织相关联，从而使他们有权在网络通道中获得权利。例如， `Isabella` 可能是 MagnetoCorp 的管理员，这可能比其他用户更有特权——来自 DigiBank 的 `Balaji`。 此外，智能合约可以在使用[交易上下文](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/transactioncontext.html)的智能合约处理期间检索此身份。

另请注意，钱包不持有任何形式的现金或代币——它们持有身份。

#### 网关

 Fabric **Gateway**。最重要的是，[网关](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/gateway.html)识别一个或多个提供网络访问的 Peer 节点——在我们的例子中是 PaperNet。

网关负责使用[连接配置文件](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/connectionprofile.html)和[连接选项](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/connectionoptions.html)将交易提案发送到网络中的正确 Peer 节点

```
await gateway.connect(connectionProfile, connectionOptions);
```

- **connectionProfile**：[连接配置文件](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/connectionprofile.html)的文件系统位置，用于将一组 Peer 节点标识为 PaperNet 的网关
- **connectionOptions**：一组用于控制 `issue.js` 与 PaperNet 交互的选项

```
channels:
  papernet:
    peers:
      peer1.magnetocorp.com:
        endorsingPeer: true
        eventSource: true

      peer2.digibank.com:
        endorsingPeer: true
        eventSource: true

peers:
  peer1.magnetocorp.com:
    url: grpcs://localhost:7051
    grpcOptions:
      ssl-target-name-override: peer1.magnetocorp.com
      request-timeout: 120
    tlsCACerts:
      path: certificates/magnetocorp/magnetocorp.com-cert.pem

  peer2.digibank.com:
    url: grpcs://localhost:8051
    grpcOptions:
      ssl-target-name-override: peer1.digibank.com
    tlsCACerts:
      path: certificates/digibank/digibank.com-cert.pem
```

MagnetoCorp 拥有 `peer1.magenetocorp.com`，DigiBank 拥有 `peer2.digibank.com`，两者都有背书节点的角色。通过 `peers:` 键链接到这些 Peer 节点，其中包含有关如何连接它们的详细信息，包括它们各自的网络地址。

#### 网络通道

应用程序可以通过连接到多个网关 Peer 节点来加入**网络中的网络**，每个网关 Peer 节点都连接到多个网络通道。 根据 `gateway.connect()` 提供的钱包标识，应用程序将在不同的通道中拥有不同的权限。

#### 构造请求

```
const contract = await network.getContract('papercontract', 'org.papernet.commercialpaper');
```

提供名称：`papercontract`。以及可选的合约命名空间： `org.papernet.commercialpaper`！

#### 提交交易

```
const issueResponse = await contract.submitTransaction('issue', 'MagnetoCorp', '00001', '2020-05-31', '2020-11-30', '5000000');
```

了解 `submitTransaction()` 参数如何与交易请求匹配。它们的值将传递给智能合约中的 `issue()` 方法，并用于创建新的商业票据。回想一下它的签名：

```
async issue(ctx, issuer, paperNumber, issueDateTime, maturityDateTime, faceValue) {...}
```

`submitTransaction` API 包含了监听交易提交的一个流程。监听提交是必须的，因为如果没有它的话，你将不会知道你的交易是否被成功地排序了，验证了并且提交到了账本上。

#### 处理响应

```
return paper.toBuffer();
```

新`票据`需要在返回到应用程序之前转换为缓冲区。请注意 `issue.js` 如何使用类方法 `CommercialPaper.fromBuffer()` 将响应缓冲区重新转换为商业票据：

与交易提案一样，智能合约完成后，应用程序可能会很快收到控制权，但事实并非如此。SDK 负责管理整个共识流程，并根据`策略`连接选项在应用程序完成时通知应用程序。 请阅读详细的[交易流程](https://hyperledger-fabric.readthedocs.io/zh_CN/txflow.html)。