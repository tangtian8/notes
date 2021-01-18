## Sofa服务接入soul网关 

### 启动SoulAdmin

### 启动SoulBootstrap

#### 修改SoulBootstrap配置文件

因为接入的是Sofa请求，所有需要增加关于Soul结合Sofa的依赖

```
        <dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>sofa-rpc-all</artifactId>
            <version>5.7.6</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-sofa</artifactId>
            <version>${project.version}</version>
        </dependency>
```

启动服务

可查看日志，加载的插件列表如下，可以看到[sofa]插件已经加载

```
2021-01-18 19:17:11.701  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[global] [org.dromara.soul.plugin.global.GlobalPlugin]
2021-01-18 19:17:11.701  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[sign] [org.dromara.soul.plugin.sign.SignPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[waf] [org.dromara.soul.plugin.waf.WafPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[rate_limiter] [org.dromara.soul.plugin.ratelimiter.RateLimiterPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[hystrix] [org.dromara.soul.plugin.hystrix.HystrixPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[resilience4j] [org.dromara.soul.plugin.resilience4j.Resilience4JPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.DividePlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[webClient] [org.dromara.soul.plugin.httpclient.WebClientPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.websocket.WebSocketPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[sofa-body-param] [org.dromara.soul.plugin.sofa.param.BodyParamPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[sofa] [org.dromara.soul.plugin.sofa.SofaPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[monitor] [org.dromara.soul.plugin.monitor.MonitorPlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.httpclient.response.WebClientResponsePlugin]
2021-01-18 19:17:11.702  INFO 1283 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.sofa.response.SofaResponsePlugin]
```

简单分析一下加载插件的过程

定位到SoulConfiguration类

通过注解@Configuration可知 这是一个配置类

```
@Configuration
@ComponentScan("org.dromara.soul")
@Import(value = {ErrorHandlerConfiguration.class, SoulExtConfiguration.class, SpringExtConfiguration.class})
@Slf4j
```

加载了SoulWebHandler类到spring bean 里面

```
    @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
```

### 启动soul.examples.sofa

查处启动日志出现了error

```
2021-01-18 19:36:55.899 ERROR 1429 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register error: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/findById","pathDesc":"Find by Id","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findById","ruleName":"/sofa/findById","parameterTypes":"java.lang.String","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
2021-01-18 19:36:56.007 ERROR 1429 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register error: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/insert","pathDesc":"Insert a row of data","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"insert","ruleName":"/sofa/insert","parameterTypes":"org.dromara.soul.examples.dubbo.api.entity.DubboTest","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
```

这两个服务注册失败

无意中发现SoulAdmin 控制台出现错误

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610970114944-ac9a17e1-b8a5-45ff-9d19-54d05e6320da.png)

点击进入SoulClientRegisterServiceImpl.java

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610970181451-bfdf069e-9086-4c15-88b9-d1668f94bc2a.png)

这是一个注册sofa服务的方法，metaDataMapper.findByServiceNameAndMethod(dto.getServiceName(), dto.getMethodName())抛出了异常，通过打断点发现

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610970963477-1c781a35-31e5-4f33-b82f-6c4f93445d92.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610970984023-19db9149-fff6-4c9a-a961-ccea40daf756.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610971026070-80c3b15f-c96f-4c49-bf2b-1ba7def45726.png)

是通过这两个参数查出的数据大于1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610971101760-7e63ece7-50be-48b1-929c-b6159bf8a66b.png)

在soulAdmin控制台修改methodName以及serviceName

防止这个数据大于1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610970891341-ae1bedc3-8e82-4aa6-8851-cb1cbb52ef96.png?x-oss-process=image%2Fresize%2Cw_1500)

把重复数据修改后再次启动

```
2021-01-18 19:58:58.310  INFO 1615 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/findById","pathDesc":"Find by Id","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findById","ruleName":"/sofa/findById","parameterTypes":"java.lang.String","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
2021-01-18 19:58:58.576  INFO 1615 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/findAll","pathDesc":"Get all data","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findAll","ruleName":"/sofa/findAll","parameterTypes":"","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
2021-01-18 19:58:58.838  INFO 1615 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/insert","pathDesc":"Insert a row of data","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"insert","ruleName":"/sofa/insert","parameterTypes":"org.dromara.soul.examples.dubbo.api.entity.DubboTest","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
```

服务注册成功。

### 验证sofa通过soul网关访问服务

前提需要开启插件

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610971366819-34274706-bc12-416d-89a3-c1f46c9a6b67.png?x-oss-process=image%2Fresize%2Cw_1500)

通过 访问 http://localhost:9195/sofa/findById?id=a

返回结果500

```
{
    "code": 500,
    "message": "Internal Server Error"
}
```

查看控制台

SoulBootstrap

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610971822463-3df979d4-4045-43c7-a657-8c929f531290.png)

发现sofaParamResolveService 为null 。猜测应该没有注入进去

查找实现类sofaParamResolveService的实现类发现只有下面的一种方法而且在测试类中

```
    static class SofaParamResolveServiceImpl implements SofaParamResolveService {

        @Override
        public Pair<String[], Object[]> buildParameter(final String body, final String parameterTypes) {
            return new ImmutablePair<>(LEFT, RIGHT);
        }
    }
```

猜测SofaParamResolveService目前没有实现类

看了看gitlab issue 已提过https://github.com/dromara/soul/issues/1001

通过不用参数测试一下通不通http://localhost:9195/sofa/findAll

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610974230605-f2594c77-8978-42c4-80ef-4d2648312d4d.png?x-oss-process=image%2Fresize%2Cw_1500)

说明sofa连接soul是通的

仔细看控制台日志，通过控制台日志分析程序调用顺序，以及错误信息。