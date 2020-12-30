# Fabric2.2.0商业票据相关概念

## 合约名称

链码（Chaincode）是一种用于部署代码到 Hyperledger Fabric 区块链网络中的通用容器。链码中定义一个或多个相关联的智能合约。每个智能合约在链码中有一个唯一的标识名。应用程序通过合约名称去访问已经实例化的链码内指定的智能合约。

### 链码

智能合约（Smart Contract）可以在链码容器中定义智能合约。当一个链码被安装到 Peer 节点并部署到通道后，链码内所有的智能合约对你的应用来说都是可用的。

### 命名

链码里的每一个智能合约都通过合约名称被唯一标识。当智能合约可以在构造类时显示分配这个名称，或者让 `Contract` 类隐式分配一个默认名称。

通过链码和智能合约的名字的组合，一个交易被唯一地定义在通道内。

### 应用程序

一旦链码安装在一个 Peer 节点而且部署在一个通道上，链码里的智能合约对于应用程序来说是可访问的

### 默认合约

被定义在链码内的第一个智能合约被成为*默认*合约。这个默认是有用的，因为链码内往往有一个被定义的智能合约；这个默认的智能合约允许应用程序直接地访问这些交易，而不需要特殊指定合约名称。

一个默认地智能合约是第一个被定义在链码的智能合约。

在大多数情况下，一个链码仅仅只包括一个单一的智能合约，所以对链码仔细命名，能够降低开发者将链码视为概念来关注的需求。在[上述](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/contractname.html#默认合约)代码例子中，感觉 `papercontract` 像是一个智能合约。

总而言之，在一个给定的链码内，合约名称是一些简单的机制去标识独自的智能合约。合约名称让应用程序更加容易的发现特定的智能合约而且更方便地使用它访问账本。

## 链码命名空间

链码命名空间允许其保持自己的世界状态与其他链码的不同。具体来说，同链码的智能合约对相同的世界状态都拥有直接的访问权，而不同链码的智能合约无法直接访问对方的世界状态。如果一个智能合约需要访问其他链码的世界状态，可通过执行链码-对-链码的调用来完成。最后，区块链可以包含涉及了不同世界状态的交易。

### 通道

如果一个节点加入了多个通道，那么将会为每个通道创建、管理一条新的区块链。而且每当有链码部署在一条新通道上，就会为其创建一个新的世界状态数据库。这就意味着通道还可以随着链码命名空间来为世界状态生成一种命名空间。

然而，相同的节点和链码容器流程可同时加入多个通道，与区块链和世界状态数据库不同，这些流程不会随着加入的通道数而增加。

例如，如果链码 `papers` 和 `bonds` 在一个新的通道上被部署了，将会有一个完全不同的区块链和两个新的世界状态数据库被创建出来。但是，节点和链码容器不会随之增加；它们只会各自连接到更多相应的通道上。

##  交易上下文Transaction context

事务上下文执行两个功能。首先，它允许开发人员在智能合约中跨事务调用定义和维护用户变量。其次，它提供了对范围广泛的Fabric api的访问，允许智能合约开发人员执行与详细事务处理相关的操作。从查询或更新分类账（不可变区块链和可修改的世界状态）到检索提交交易的应用程序的数字标识。

当智能合约部署到一个通道并使其可用于每个后续的事务调用时，就会创建一个事务上下文。事务上下文帮助智能合约开发人员编写功能强大、效率高且易于推理的程序。

## 交易处理器

交易处理器允许智能合约开发人员在应用程序和智能合约交互期间的关键点上定义通用处理。交易处理器是可选的，但是如果定义了，它们将在调用智能合约中的每个交易之前或之后接收控制权。还有一个特定的处理器，当请求调用未在智能合约中定义的交易时，该处理程序接收控制。

这里是一个[商业票据智能合约示例](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/smartcontract.html)的交易处理器的例子：

![develop.transactionhandler](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/_images/develop.diagram.2.png)

前置、后置和未知交易处理器。在这个例子中，`beforeTransaction()` 在 **issue**, **buy** 和 **redeem** 交易之前被调用。`afterTransaction()` 在 **issue**, **buy** 和 **redeem** 交易之后调用。`UnknownFunction()` 当请求调用未在智能合约中定义的交易时调用。（通过不为每个交易重复 `beforeTransaction` 和 `afterTransaction` 框来简化该图）

#### 定义处理器

```
CommercialPaperContract extends Contract {

    ...

    async beforeTransaction(ctx) {
        // Write the transaction ID as an informational to the console
        console.info(ctx.stub.getTxID());
    };

    async afterTransaction(ctx, result) {
        // This handler interacts with the ledger
        ctx.stub.cpList.putState(...);
    };

    async unknownTransaction(ctx) {
        // This handler throws an exception
        throw new Error('Unknown transaction function');
    };

}
```

## 背书策略

背书策略定义了要背书一项交易使其生效所需要的最小组织集合。要想对一项交易背书，组织的背书节点需要运行与该交易有关的智能合约，并对结果签名。当排序服务将交易发送给提交节点，节点们将各自检查该交易的背书是否满足背书策略。如果不满足的话，交易被撤销，不会对世界状态产生影响。

背书策略从以下两种不同的维度来发挥作用：既可以被设置为整个命名空间，也可被设置为单个状态键。它们是使用诸如 `AND` 和 `OR` 这样的逻辑表述来构成的。例如，在 PaperNet 中可以这样来用： MagnetoCorp 将一篇论文卖给 DigiBank，其背书策略可被设定为 `AND(MagnetoCorp.peer, DigiBank.peer)`，这就要求任何对该论文做出的改动需被 MagnetoCorp 和 DigiBank 同时背书。

## 连接配置文件

连接配置文件描述了一组组件，包括 Hyperledger Fabric 区块链网络中的 Peer 节点、排序节点以及 CA。它还包含与这些组件相关的通道和组织信息。连接配置文件主要由应用程序用于配置处理所有网络交互的[网关](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/gateway.html)，从而使其可以专注于业务逻辑。连接配置文件通常由了解网络拓扑的管理员创建。

## 连接选项

```
const connectionOptions = {
  identity: userName,
  wallet: wallet,
  eventHandlerOptions: {
    commitTimeout: 100,
    strategy: EventStrategies.MSPID_SCOPE_ANYFORTX
    }
  };
```

![profile.scenario](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/_images/develop.diagram.35.png)

- `wallet` 标识了将要被应用程序上的网关所使用的钱包。看一下交互**1**，钱包是由应用程序指定的，但它实际上是检索身份的网关。

  一个钱包必须被指定；最重要的决定是钱包使用的[类型](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/developapps/wallet.html#type)，是文件系统、内存、HSM 还是数据库。

- `identity` 应用程序从 `wallet` 中使用的用户身份。看一下交互**2a**；用户身份被应用程序指定，也代表了应用程序的用户 Isabella，**2b**。身份实际上被网络检索。

  在我们的例子中，Isabella 的身份将会被不同的 MSP（**2c**，**2d**）使用，用于确定他为来自 MagnetoCorp 的一员，并且在其中扮演着一个特殊的角色。这两个事实将相应地决定了他在资源上的许可。

  一个用户的身份必须被指定。正如你所看到的，这个身份对于 Hyperledger Fabric 是一个*有权限的*网络的概念来说是基本的原则——所有的操作者有一个身份，包括应用程序、Peer 节点和排序节点，这些身份决定了他们在资源上的控制。你可以在成员身份服务[话题](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/membership/membership.html)中阅读更过关于这方面的概念。

- `clientTIsIdentity` 是可以从钱包（**3a**）获取到的身份，用于确保网关和不同的t通道组件之间的交流（**3b**），比如 Peer 节点和排序节点。

  注意：这个身份不同于用户身份。即使 `clientTlsIdentity` 对于安全通信来说很重要，但它并不像用户身份那样基础，因为它的范围没有超过确保网络的通信。

  `clientTlsIdentity` 是可选项。建议你把它设置进生产环境中。你应该也使用不同的 `clientTlsIdentity` 用做 `identity`，因为这些身份有着非常不同的意义和生命周期。例如，如果 `clientTIsIdentity` 被破坏，那么你的 `identity` 也会被破坏；让他们保持分离会更加安全。

- `eventHandlerOptions.commitTimeout` 是可选的。它以秒为单位指定网关在将控制权返回给应用程序之前，应该等待任何对等方提交事务的最大时间量(**4a**)。用于通知的 Peer 节点集合由 `eventHandlerOptions` 选项决定。如果没有指定 commitTimeout，网关默认将使用300秒的超时。

- `eventHandlerOptions.strategy` 是可选的。它定义了网关用来监听交易被提交通知的 Peer 节点集合。例如，是监听组织中的单一的 Peer 节点还是全部的节点。可以采用以下参数之一：

  - `EventStrategies.MSPID_SCOPE_ANYFORTX` 监听用户组织内的**一些** Peer 节点。在我们的例子中，看一下交互点**4b**；MagnetoCorp 中的 peer1、peer2、peer3 节点能够通知网关。

  - `EventStrategies.MSPID_SCOPE_ALLFORTX` **这是一个默认值**。监听用户组织内的**所有** Peer 节点。在我们例子中，看一下交互点**4b**。所有来自 MagnetoCorp 的节点必须全部通知网关：peer1、peer2 和 peer3 节点。只有已知/被发现和可用的 Peer 节点才会被计入，停止或者失效的节点不包括在内。

  - `EventStrategies.NETWORK_SCOPE_ANYFORTX` 监听整个网络通道内的**一些** Peer 节点。在我们的例子中，看一下交互点 **4b** 和 **4c**；MagnetoCorp 中部分 Peer 节点1-3 或者 DigiBank 的部分 Peer 节点 7-9 能够通知网关。

  - `EventStrategies.NETWORK_SCOPE_ALLFORTX` 监听整个网络通道内**所有** Peer 节点。在我们的例子中，看一些交互 **4b** 和 **4c**。所有来自 MagnetoCorp 和 DigiBank 的 Peer 节点 1-3 和 Peer 节点 7-9 都必须通知网关。只有已知/被发现和可用的 Peer 节点才会被计入，停止或者失效的节点不包括在内。

  - <`PluginEventHandlerFunction`> 用户定义的事件处理器的名字。这个允许用户针对事件处理而定义他们自己的逻辑。看一下如何[定义](https://hyperledger.github.io/fabric-sdk-node/master/tutorial-transaction-commit-events.html)一个时间处理器插件，并检验一个[示例处理器](https://github.com/hyperledger/fabric-sdk-node/blob/master/test/integration/network-e2e/sample-transaction-event-handler.js)。

    如果你有一个迫切需要事件处理的需求的话，那么用户定义的事件处理器是有必要的；大体上来说，一个内建的事件策略是足够的。一个用户定义的事件处理器的例子可能是等待超过半数的组织内的 Peer 节点，以去确认交易被提交。

  如果你确实指定了一个用户定义的事件处理器，她不会影响你的一个应用程序的逻辑；它是完全独立的。处理器被处理过程中的SDK调用；它决定了什么时候调用它，并且使用它的处理结果去选择哪个 Peer 节点用于事件通知。当SDK完成它的处理的时候，应用程序收到控制。

  如果一个用户定义的事件处理器没有被指定，那么 `EventStrategies`的默认值将会被使用。

- `discovery.enabled` 是可选项，可能的值是 `true` 还是 `false`。默认是 `ture`。它决定了网关是否使用[服务发现](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/discovery-overview.html)去增加连接配置文件里指定的网络拓扑。看一下交互点**6**；peer 节点的 gossip 信息会被网关使用。

  这个值会被 `INITIALIIZE-WITH-DISCOVERY` 的环境变量覆盖，其值可以被设置成 `true` 或者 `false`。

- `discovery.asLocalhost` 是可选项，可能的值是 `true`或者 `false`。默认是`true`。它决定在服务发现期间发现的 IP 地址是否从 docker 网络传递给本地主机。

  通常，开发人员将为其网络组件（如 Peer 节点、排序节点 和 CA 节点）编写使用 docker 容器的应用程序，但是他们自己本身不会运行在 docker 容器内。这个就是为什么 `true` 是默认的，这生产环境中，应用程序很可能以网络组件相同的方式运行在 docker 中，因此不需要地址转换。在这种情况下，应用程序应该要不明确指定 `false`，要不使用环境变量来覆盖。

## 钱包

一个钱包包括了一组用户身份。当与通道连接的时候，应用程序会从这些身份中选择一个用户来运行。对通道资源的访问权限，比如账本，由与 MSP（Membership provider）相关联的这个身份所定义。

钱包能够让一个用户拥有多个身份，并且每一个身份都能由不同的 CA 颁发。

*三种不同的钱包类型:文件系统、内存和 CouchDB*

- **文件系统（FileSystem）**：这是存储钱包最常见的地方；文件系统是无处不在的、容易理解且可以挂载在网络上。对于钱包来说，这是很好的默认选择。
- **内存（In-memory）**：存储在应用程序里的钱包。当你的应用程序正在运行在一个没有访问文件系统的约束环境的时候，使用这种类型的钱包；有代表性的是 web 浏览器。需要记住的是这种类型的钱包是不稳定的；在应用程序正常结束或者崩溃的时候，身份将失去丢失。
- **CouchDB**：存储在 Couch DB 的钱包。这是最罕见的一种钱包存储形式，但是对于想去使用数据备份和恢复机制的用户来说，CouchDB 钱包能够提供一个有用的选择去简化崩溃的恢复

## 网关

