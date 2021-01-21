# soul使用websocket实现数据同步

## 官网架构图分析

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611055756278-22e6aecb-512c-4676-af39-bbbce1ae8416.png)

## WebSocket数据同步

### soulAdmin操作数据的时候

在souladmin 中 有个DataChangedEvent类他继承了ApplicationEvent

```
public class DataChangedEvent extends ApplicationEvent {}
```

这个事件是在操作完数据库后触发的,可以看见很多service里面用到了DataChangedEvent

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611057170483-bd912cbf-f445-45e6-882f-441e7e189540.png?x-oss-process=image%2Fresize%2Cw_1500)

通过DataChangedEventDispatcher 类 该类继承了ApplicationListener类来监听触发的事件，监听的事件类型包括以下数据的变换

```
   public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean { 
     @Override
     @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH://App auth config group enum
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN://Plugin config group enum.
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE://Rule config group enum.
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR://Selector config group enum
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA://Meta data config group enum.
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }
   }
```

从上面可知DataChangedListener的实例是listener，进入DataChangedListener，发现是一个接口

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611057481740-1264758c-c5b5-434f-a05d-c4c42b88c82b.png?x-oss-process=image%2Fresize%2Cw_1500)

里面有很多数据同步的实现。

这里我们只分析Websocket

以上分析我们可以得知：

数据库更新 -> 通知事件 -> 监听到事件 -> 执行数据同步（http长轮询，zk，webSocket等）

### SoulBootstrap启动时候

在 SoulBootstrap 的pom文件中发现有soul-spring-boot-starter-sync-data-websocket依赖

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-websocket</artifactId>
            <version>${project.version}</version>
        </dependency>
```

进入soul-spring-boot-starter-sync-data-websocket发现

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-sync-data-websocket</artifactId>
            <version>${project.version}</version>
        </dependency>
```

进入soul-sync-data-websocket项目SoulWebsocketClient中



发现了 send(DataEventTypeEnum.MYSELF.name()); 这个是通知admin加载所有的admin 数据库中所有的数据

```
    @Override
    public void onOpen(final ServerHandshake serverHandshake) {
        if (!alreadySync) {
            send(DataEventTypeEnum.MYSELF.name());
            alreadySync = true;
        }
    }
```

在soulAdmin中 onMessage方法发现 if (message.equals(DataEventTypeEnum.MYSELF.name())) 就加载所有数据

```
    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
        appAuthService.syncData();
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        List<SelectorData> selectorDataList = selectorService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
        List<RuleData> ruleDataList = ruleService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
        metaDataService.syncData();
        return true;
    }
```

所以当启动SoulBootstrap时候

SoulBootstrap->soul-spring-boot-starter-sync-data-websocket->soul-sync-data-websocket-> send(DataEventTypeEnum.MYSELF.name())->onMessage(syncAll)->通知事件 -> 监听到事件 -> 执行数据同步（http长轮询，zk，webSocket等）

### soul-examples-sofa启动时候

大致分析了源码 

流程是 通过@SoulSofaClient拦截到URL->通过OkHttpTools工具把数据提交到admin->admin在同步数据