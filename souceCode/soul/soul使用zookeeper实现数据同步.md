# soul使用zookeeper实现数据同步

### zookeeper介绍

Zookeeper是一个Apache开源的分布式的应用，为系统架构提供协调服务。从设计模式角度来审视：该组件是一个基于观察者模式设计的框架，负责存储和管理数据，接受观察者的注册，一旦数据的状态发生变化，Zookeeper就将负责通知已经在Zookeeper上注册的观察者做出相应的反应，从而实现集群中类似Master/Slave管理模式。ZooKeeper的目标就是封装好复杂易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户。

ZooKeeper是一个分布式服务框架，是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等

ZooKeeper是一个树形结构的目录服务，支持变更推送

在ZooKeeper中，节点分为两类：

　　机器节点：

　　　　指构成集群的机器

　　数据节点ZNode：

　　　　指数据模型中的数据单元　　

　　　　ZooKeeper将所有数据存储在内存中，数据模型是一棵树(ZNode Tree)，由斜杠（/）进行分割的路径，就是一个ZNode，例如/services/customer

　　　　每个ZNode上都会保存自己的数据内容，同时还会保存一系列属性信息

　　　　Znode可分为：

　　　　　　持久节点：指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在ZooKeeper上

　　　　　　临时节点：它的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除

### soulAdmin中的zk

#### 引入zk需要的依赖

```
<dependency>
                <groupId>com.101tec</groupId>
                <artifactId>zkclient</artifactId>
                <exclusions>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>slf4j-log4j12</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>log4j</groupId>
                        <artifactId>log4j</artifactId>
                    </exclusion>
                </exclusions>
  </dependency>
```

#### zk的配置文件

```
soul:
  sync:
      zookeeper:
        url: 120.79.218.62:2181
        sessionTimeout: 5000
        connectionTimeout: 2000
```

#### 通过事件触发zk

在soul-admin中数据的更新触发了事件。事件再去使用DataChangedListener来同步数据。DataChangedListener是一个接口。其中包括了zookeeper的实现。



```
public class ZookeeperDataChangedListener implements DataChangedListener {

    private final ZkClient zkClient;

    public ZookeeperDataChangedListener(final ZkClient zkClient) {
        this.zkClient = zkClient;
    }

    @Override
    public void onAppAuthChanged(final List<AppAuthData> changed, final DataEventTypeEnum eventType) {
            ...
            // create or update
            upsertZkNode(appAuthPath, data);
        
    }

    @SneakyThrows
    @Override
    public void onMetaDataChanged(final List<MetaData> changed, final DataEventTypeEnum eventType) {
            ...
            // create or update
            upsertZkNode(metaDataPath, data);
            ...
    }

    @Override
    public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
            ...
            upsertZkNode(pluginPath, data);
            ...
    }

    @Override
    public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
            ···
            createZkNode(selectorParentPath);
            //create or update
            upsertZkNode(selectorRealPath, data);
            ···
    }

    @Override
    public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
       ···
    }
    
    private void createZkNode(final String path) {
        ···
    }
    
    private void upsertZkNode(final String path, final Object data) {
        ···
    }
```

### soulBootstrap中的zk

引入zk同步的依赖

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
            <version>${project.version}</version>
        </dependency>
```

分析soul-spring-boot-starter-sync-data-zookeeper

改starter项目为zk增加了配置文件并引入soul-sync-data-zookeeper项目

分析soul-sync-data-zookeeper

```
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {

    private final ZkClient zkClient;
}
```

看懂ZookeeperSyncDataService需要补充一下 ZKClient 的知识https://blog.csdn.net/u012988901/article/details/83753496

#### 出现的问题

启动soulBootstrap后MetaDataCache类为null，可能数据没有同步到内存中

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611144603507-29b48456-6395-4dc2-a56d-a3b115dea657.png)

数据是同步到ConcurrentMap里面

```
private static final ConcurrentMap<String, MetaData> META_DATA_MAP = Maps.newConcurrentMap();
```

那就从MetaDataCache类中put方法找起，为什么没有put进去

```
    public void cache(final MetaData data) {
        META_DATA_MAP.put(data.getPath(), data);
    }
```

该方法调用了cache方法增加MetaData 数据

```
    private void cacheMetaData(final MetaData metaData) {
        Optional.ofNullable(metaData).ifPresent(data -> metaDataSubscribers.forEach(e -> e.onSubscribe(metaData)));
    }
```



往下走

```
    private void watchMetaData() {
        final String metaDataPath = ZkPathConstants.META_DATA;
        List<String> childrenList = zkClientGetChildren(metaDataPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(metaDataPath, children);
                cacheMetaData(zkClient.readData(realPath));
                subscribeMetaDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.APP_AUTH, metaDataPath, childrenList);
    }
```

发现 List<String> childrenList = zkClientGetChildren(metaDataPath); 为null

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611145484648-12c8c1a4-fb38-4c9c-bef1-a271ff0e8976.png)

经过排查

soulAdmin配置文件zk信息没有配置正确



问题二

启动soulBootstrap后报空指针异常

正在排查中



定位这个问题报错

```
    private void watchMetaData() {
        final String metaDataPath = ZkPathConstants.META_DATA;
        List<String> childrenList = zkClientGetChildren(metaDataPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(metaDataPath, children);
                cacheMetaData(zkClient.readData(realPath));
                subscribeMetaDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.APP_AUTH, metaDataPath, childrenList);
    }
```

发现返回的数据编码有无，猜测是这个问题

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611186259082-25905737-9325-425b-84b1-3c4d8cf324f0.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611188298615-a2fbbac6-f649-49f1-8d0a-83a202601c53.png)

最终的解决方案

重启启动了SoulAdmin发现加载了以前没有加载的数据

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611188778744-a337e003-33a5-4c3c-bc44-3697ff5b9aae.png)

再次重新启动soulBootstrap 和 sofaTest 最终通过zk同步成功完成

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611188861831-bbe91e22-fc9f-4e60-a4c1-d0cd61e92a1b.png)

后面需要学习zk的机制