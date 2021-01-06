# Fabric2.2.0将智能合约部署到通道

# 创建网络

The Hyperledger Fabric 基础网络的组成部分包括一个节点及该节点的账本数据库，一个排序服务和一个证书授权中心。以上每个组件都在一个 Docker 容器中运行。

peer0.org1.example.com 属于 DigiBank 公司
peer0.org2.example.com 属于 MagnetoCorp 公司

## 以 MagnetoCorp 的身份安装和批准智能合约

以 MagnetoCorp 的身份管理网络
//进入到当前组织

```
cd commercial-paper/organization/magnetocorp
```

//该脚本设置环境变量

``` 
source magnetocorp.sh 
```

//（(magnetocorp admin)）组织将智能合约打包成连码。// js 版本

```
(magnetocorp admin)$ peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```

//这个（magnetocorp admin）管理员就可以安装了

```
(magnetocorp admin)$ peer lifecycle chaincode install cp.tar.gz
```

//结果（安装完成后就看到如下信息）
2020-01-30 18:32:33.762 EST [cli.lifecycle.chaincode] submitInstallProposal -> INFO 001 Installed remotely: response:<status:200 payload:"\nEcp_0:ffda93e26b183e231b7e9d5051e1ee7ca47fbf24f00a8376ec54120b1a2a335c\022\004cp_0" >
2020-01-30 18:32:33.762 EST [cli.lifecycle.chaincode] submitInstallProposal -> INFO 002 Chaincode code package identifier: cp_0:ffda93e26b183e231b7e9d5051e1ee7ca47fbf24f00a8376ec54120b1a2a335c

//因为magneticorp管理员已经设置了CORE_PEER_地址=本地主机：9051到将它的命令定向到peer0.org2。example.com网站，INFO 001远程安装。。。指示papercontract已成功安装在此对等机上
//安装智能合约后，我们需要批准papercontract的链码定义为MagnetorCorp。第一步是找到我们在对等机上安装的链码的packageID。我们可以使用peer lifecycle chaincode queryinstalled命令查询packageID

```
peer lifecycle chaincode queryinstalled
```

（结果）
Installed chaincodes on peer:
Package ID: cp_0:ffda93e26b183e231b7e9d5051e1ee7ca47fbf24f00a8376ec54120b1a2a335c, Label: cp_0

//在下一步中，我们需要包ID，因此我们将它保存为环境变量。对于所有用户，包ID可能不相同，因此您需要使用从命令窗口返回的包ID来完成此步骤

```
export PACKAGE_ID=cp_0:****
```

//以下命令管理员现在可以使用以下命令批准MagnetoCorp的链码定义

```
(magnetocorp admin)$ peer lifecycle chaincode approveformyorg --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name papercontract -v 0 --package-id $PACKAGE_ID --sequence 1 --tls --cafile $ORDERER_CA
```


使用链码定义时，频道成员需要同意的最重要的链码参数之一是链码认可策略。背书策略描述了在确定交易有效之前必须背书（执行和签署）交易的一组组织。通过批准没有--policy标志的papercontract链码，magneticorp管理员同意使用该频道的默认认可策略，在mychannel测试频道的情况下，该策略要求频道上的大多数组织认可一项交易。所有交易，无论有效还是无效，都将记录在分类账区块链上，但只有有效的交易才会更新世界状态。

## 以 DigiBank 的身份安装和批准智能合约

```
(digibank admin)$ cd commercial-paper/organization/digibank/
```


//设置环境变量

```
source digibank.sh
```


//打包

```
(digibank admin)$ peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```

//安装

```
(digibank admin)$ peer lifecycle chaincode install cp.tar.gz
```


//然后我们需要查询并保存刚刚安装的链码的packageID

```
(digibank admin)$ peer lifecycle chaincode queryinstalled
```

//将包ID另存为环境变量。使用从控制台返回的包ID完成此步骤

```
export PACKAGE_ID=cp_0:ffda93e26b183e231b7e9d5051e1ee7ca47fbf24f00a8376ec54120b1a2a335c
```

//Digibank管理员现在可以批准papercontract的链码定义

```
(digibank admin)$ peer lifecycle chaincode approveformyorg --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name papercontract -v 0 --package-id $PACKAGE_ID --sequence 1 --tls --cafile $ORDERER_CA
```

## 将链码定义提交到通道

既然DigiBank和magneticorp都已经批准了papernet链码，那么我们就有了将链码定义提交给频道所需的大部分（2/2）。一旦在频道上成功地定义了链码，则该频道上的客户端应用程序就可以调用papercontract链码中的商业纸张智能合约。由于任何一个组织都可以向频道提交链码，因此我们将继续以DigiBank管理员的身份运行
//在DigiBank管理员将papercontract链代码的定义提交给通道后，将创建一个新的Docker chaincode容器，以便在两个PaperNet对等机上运行papercontract
//DigiBank管理员使用peer lifecycle chaincode commit命令将papercontract的链码定义提交到mychannel：

```
(digibank admin)$ peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} --channelID mychannel --name papercontract -v 0 --sequence 1 --tls --cafile $ORDERER_CA --waitForEvent
```



结果（chaincode容器将在chaincode定义提交到通道之后启动。您可以使用docker ps命令查看papercontract容器在两个对等机上启动）
(digibank admin)$ docker ps

CONTAINER ID        IMAGE                                                                                                                                                               COMMAND                  CREATED             STATUS              PORTS                                        NAMES
d4ba9dc9c55f        dev-peer0.org1.example.com-cp_0-ebef35e7f1f25eea1dcc6fcad5019477cd7f434c6a5dcaf4e81744e282903535-05cf67c20543ee1c24cf7dfe74abce99785374db15b3bc1de2da372700c25608   "docker-entrypoint.s…"   30 seconds ago      Up 28 seconds                                                    dev-peer0.org1.example.com-cp_0-ebef35e7f1f25eea1dcc6fcad5019477cd7f434c6a5dcaf4e81744e282903535
a944c0f8b6d6        dev-peer0.org2.example.com-cp_0-1487670371e56d107b5e980ce7f66172c89251ab21d484c7f988c02912ddeaec-1a147b6fd2a8bd2ae12db824fad8d08a811c30cc70bc5b6bc49a2cbebc2e71ee   "docker-entrypoint.s…"   31 seconds ago      Up 28 seconds                                                    dev-peer0.org2.example.com-cp_0-1487670371e56d107b5e980ce7f66172c89251ab21d484c7f988c02912ddeaec