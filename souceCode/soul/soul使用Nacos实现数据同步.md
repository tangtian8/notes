## Nacos

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

https://nacos.io/zh-cn/docs/what-is-nacos.html

Nacos后台管理界面

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611322946922-4856c709-ce64-4239-8eba-831832626f14.png)

## soul-admin中使用Nacos

引入pom文件

```
    <properties>
        <nacos-client.version>1.2.0</nacos-client.version>
    </properties>        
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>${nacos-client.version}</version>
        </dependency>
```

yml文件

```
soul:
  sync:
    nacos:
      url: 120.79.218.62:8848
      namespace: 1ba37a07-be48-488d-ade3-852d293ae93c
      acm:
        enabled: false
        endpoint: acm.aliyun.com
        namespace:
        accessKey:
        secretKey:
```

NacosDataChangedListener 是DataChangedListener的实现

DataChangedListener引入了ConfigService，并在构造函数中初始化了它

```
    private final ConfigService configService;
    public NacosDataChangedListener(final ConfigService configService) {
        this.configService = configService;
    }
```

ConfigService接口如下方法,这些方法将把配置信息提交到Nacos。

```
public interface ConfigService {
    String getConfig(String var1, String var2, long var3) throws NacosException;// 根据dataid、group获取配置信息
    String getConfigAndSignListener(String var1, String var2, long var3, Listener var5) throws NacosException;

    void addListener(String var1, String var2, Listener var3) throws NacosException;// 针对dataid、group增加监听器

    boolean publishConfig(String var1, String var2, String var3) throws NacosException;// 发布配置

    boolean removeConfig(String var1, String var2) throws NacosException;// 删除配置

    void removeListener(String var1, String var2, Listener var3);// 服务状态

    String getServerStatus();// 关闭回收状态
}
```

## soul-bootstrap中使用Nacos 

引入pom文件

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-nacos</artifactId>
            <version>${project.version}</version>
        </dependency>
```

yml文件

```
soul :
  sync:
    nacos:
      url: 120.79.218.62:8848
      namespace: 1ba37a07-be48-488d-ade3-852d293ae93c
      acm:
        enabled: false
        endpoint: acm.aliyun.com
        namespace:
        accessKey:
        secretKey:
```

NacosSyncDataService负责soul-bootstrap数据的同步，该类同样传入ConfigService 对象，并且它是NacosCacheHandler的子类。

```
public class NacosSyncDataService extends NacosCacheHandler implements AutoCloseable, SyncDataService {

    public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        start();
    }
```

进入NacosCacheHandler，通过Listener监听数据变换

```
    protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        oc.change(getConfigAndSignListener(dataId, listener));
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }
```

获取配置的同时注册了监听器

```
    @SneakyThrows
    private String getConfigAndSignListener(final String dataId, final Listener listener) {
        return configService.getConfigAndSignListener(dataId, GROUP, 6000, listener);
    }
```