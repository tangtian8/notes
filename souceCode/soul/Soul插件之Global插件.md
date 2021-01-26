### Global插件的starter

Global插件有个starter

在soul-admin的pom文件中引入了global

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-global</artifactId>
            <version>${project.version}</version>
        </dependency>
```

上述starter 的作用经过代码发现加载了三种Bean到spring中，他们分别是SoulPlugin、MetaDataSubscriber、SoulContextBuilder

```
    @Bean
    public SoulPlugin globalPlugin(final SoulContextBuilder soulContextBuilder) {
        return new GlobalPlugin(soulContextBuilder);
    }
    @Bean
    @ConditionalOnMissingBean(value = SoulContextBuilder.class, search = SearchStrategy.ALL)
    public SoulContextBuilder soulContextBuilder() {
        return new DefaultSoulContextBuilder();
    }
    @Bean
    public MetaDataSubscriber metaDataAllSubscriber() {
        return new MetaDataAllSubscriber();
    }
```

根据类关系，GlobalPlugin插件实现了SoulPlugin，并把SoulContextBuilder放在构造方法里，说明GlobalPlugin的实例需要用到SoulContextBuilder。

### ![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611619668710-04fe0b78-8a95-4e0e-acc3-0b8d218084fb.png)

### GlobalPlugin 

#### GlobalPlugin的execute方法

改方法主要生成soulContext

其中**Upgrade**: protocols. **Upgrade 头**指定一项或多项协议名，按优先级排序，以逗号分隔。

普通http协议就用builder.build(exchange);

websocket或其他升级协议用transformMap(queryParams);

```
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
```

最后把把soulContext 放在了ServerWebExchange的实例exchange里面，这里很重要，为后面的execute提供上下文。

```
 exchange.getAttributes().put(Constants.CONTEXT, soulContext);
```

### DefaultSoulContextBuilder

该类的核心方法build，并且GlobalPlugin把DefaultSoulContextBuilder作为构造方法属性引入。

他的作用就是构建http协议请求的SoulContext

```
    @Override
    public SoulContext build(final ServerWebExchange exchange) {
        final ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        MetaData metaData = MetaDataCache.getInstance().obtain(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            exchange.getAttributes().put(Constants.META_DATA, metaData);
        }
        return transform(request, metaData);
    }
```

该类的实例做两件事情

1.把元数据放在exchange中

```
exchange.getAttributes().put(Constants.META_DATA, metaData);
```

2.对soulContext放入对应请求（Dubbo，Sofa，Tars，Http）的名字，而对应的方法就是通过metaData数据去匹配。metaData数据是应用端启动后会通知admin去同步数据。

```
 transform(request, metaData);
```

### 总结 global插件

global插件主要去生成soulContext，soulContext内容包括

```
private String module;

    /**
     * is method name .
     */
    private String method;

    /**
     * is rpcType data. now we only support "http","dubbo","springCloud","sofa".
     */
    private String rpcType;

    /**
     * httpMethod now we only support "get","post" .
     */
    private String httpMethod;

    /**
     * this is sign .
     */
    private String sign;

    /**
     * timestamp .
     */
    private String timestamp;

    /**
     * appKey .
     */
    private String appKey;

    /**
     * path.
     */
    private String path;
    
    /**
     * the contextPath.
     */
    private String contextPath;

    /**
     * realUrl.
     */
    private String realUrl;

    /**
     * this is dubbo params.
     */
    private String dubboParams;

    /**
     * startDateTime.
     */
    private LocalDateTime startDateTime;
```