# soul入门

## 官方文档

https://dromara.org/zh-cn/docs/soul/soul.html

## soul 介绍

### soul是网关

作者参考了Kong，Spring-Cloud-Gateway等优秀的网关后，实现一个基于 [WebFlux](http://www.iocoder.cn/Spring-Boot/WebFlux/?self) 异步的,高性能的,跨语言的,响应式的API网关，名字叫Soul。

### soul功能列表

- 各种语言无缝集成到 Dubbo、Spring Cloud、Spring Boot 中
- 插件化设计思想，插件热插拔,易扩展。
- 灵活的流量筛选，能满足各种流量控制。
- 内置丰富的插件支持，鉴权，限流，熔断，防火墙等等。
- 流量配置动态化，性能极高，网关消耗在 1~2ms。
- 支持集群部署，支持 A/B Test, 蓝绿发布。

## 前期准备

拉取代码到本地 

基于master 分支

```
git clone https://github.com/dromara/soul
```

导入maven,编译一下

```
mvn clean package install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true
```

### 分析大体结构

- soul-admin : 插件和其他信息配置的管理后台
- soul-bootstrap : 用于启动项目
- soul-client : 用户可以使用 Spring MVC，Dubbo，Spring Cloud 快速访问
- soul-common : 框架的通用类
- soul-dist : 构建项目
- soul-metrics : prometheus实现的 metrics
- soul-plugin : Soul 支持的插件集合
- soul-spi : 定义 Soul spi
- soul-spring-boot-starter : 支持 spring boot starter
- soul-sync-data-center : 提供 ZooKeeper，HTTP，WebSocket，Nacos 的方式同步数据
- soul-examples : RPC 示例项目
- soul-web : 包括插件、请求路由和转发等的核心处理包

## 运行soul-admin

### soul-admin项目结构

```
.
├── java
│   └── org
│       └── dromara
│           └── soul
│               └── admin
│                   ├── SoulAdminBootstrap.java
│                   ├── config
│                   ├── controller
│                   ├── dto
│                   ├── entity
│                   ├── exception
│                   ├── listener
│                   ├── mapper
│                   ├── page
│                   ├── query
│                   ├── result
│                   ├── service
│                   ├── spring
│                   ├── transfer
│                   ├── utils
│                   └── vo
└── resources
    ├── META-INF
    ├── application-h2.yml
    ├── application-local.yml
    ├── application.yml
    ├── mappers
    ├── mybatis
    └── static
```

### 不需要数据库和创建表

在当前项目的org.dromara.soul.admin.service.init包下 将自动创建soul数据库和表

```java
@Component
public class LocalDataSourceLoader implements InstantiationAwareBeanPostProcessor {
    
    private static final Logger LOGGER = LoggerFactory.getLogger(LocalDataSourceLoader.class);
    
    private static final String SCHEMA_SQL_FILE = "META-INF/schema.sql";
    
    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, final String beanName) throws BeansException {
        if (bean instanceof DataSourceProperties) {
            this.init((DataSourceProperties) bean);
        }
        return bean;
    }
    
    @SneakyThrows
    protected void init(final DataSourceProperties properties) {
     ...
        
    }
    
    private void execute(final Connection conn) throws Exception {
       ...
    }
    
}
 
```

### 修改配置文件

在resources目录下application-local.yml中

```修改自己的数据库地址和密码```

```
jdbc:mysql://120.79.218.62:3306/soul?useUnicode=true&characterEncoding=utf-8
```

### 启动项目

运行SoulAdminBootstrap类main函数启动启动

Tomcat started on port(s): 9095 (http) with context path ''

### 运行

浏览器地址

http://localhost:9095/

用户名:admin 密码123456

### 目前不懂的地方

配置文件中	

注释的配置功能是干什么？

```
  #    zookeeper:
  #        url: localhost:2181
  #        sessionTimeout: 5000
  #        connectionTimeout: 2000
  #    http:
  #      enabled: true
  #    nacos:
  #      url: localhost:8848
  #      namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
  #      acm:
  #        enabled: false
  #        endpoint: acm.aliyun.com
  #        namespace:
  #        accessKey:
  #        secretKey:
```



## 运行soul-bootstrap

个人理解该项目用于加载各个插件的启动

### soul-bootstrap项目结构

```
.
├── java
│   └── org
│       └── dromara
│           └── soul
│               └── bootstrap
│                   ├── SoulBootstrapApplication.java
│                   ├── configuration
│                   │   └── SoulNettyWebServerFactory.java
│                   └── filter
│                       └── HealthFilter.java
└── resources
    ├── application-local.yml
    └── application.yml
```

### 配置soul-admin控制台

在resources的application-local.yml中

增加soul-admin服务地址：localhost:9095

```
soul :
    sync:
        websocket :
             urls: ws://localhost:9095/websocket
```

### 启动项目

运行SoulBootstrapApplication类main函数启动启动

显示加载各种插件

```
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[global] [org.dromara.soul.plugin.global.GlobalPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[sign] [org.dromara.soul.plugin.sign.SignPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[waf] [org.dromara.soul.plugin.waf.WafPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[rate_limiter] [org.dromara.soul.plugin.ratelimiter.RateLimiterPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[hystrix] [org.dromara.soul.plugin.hystrix.HystrixPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[resilience4j] [org.dromara.soul.plugin.resilience4j.Resilience4JPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.DividePlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[webClient] [org.dromara.soul.plugin.httpclient.WebClientPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.websocket.WebSocketPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[alibaba-dubbo-body-param] [org.dromara.soul.plugin.alibaba.dubbo.param.BodyParamPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[dubbo] [org.dromara.soul.plugin.alibaba.dubbo.AlibabaDubboPlugin]
2021-01-14 23:08:29.909  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[monitor] [org.dromara.soul.plugin.monitor.MonitorPlugin]
2021-01-14 23:08:29.910  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.httpclient.response.WebClientResponsePlugin]
2021-01-14 23:08:29.910  INFO 26617 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.alibaba.dubbo.response.DubboResponsePlugin]
```

### 监控地址

通过下面日志可知

```
2021-01-14 23:20:50.629  INFO 26708 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2021-01-14 23:20:51.699  INFO 26708 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 9195
```

### 目前不懂的地方

配置文件中	

注释的配置功能是干什么？

```
#        zookeeper:
#             url: localhost:2181
#             sessionTimeout: 5000
#             connectionTimeout: 2000
#        http:
#             url : http://localhost:9095
#        nacos:
#              url: localhost:8848
#              namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
#              acm:
#                enabled: false
#                endpoint: acm.aliyun.com
#                namespace:
#                accessKey:
#                secretKey:


#eureka:
#  client:
#    serviceUrl:
#      defaultZone: http://localhost:8761/eureka/
#  instance:
#    prefer-ip-address: true

spring:
#   cloud:
#    nacos:
#       discovery:
#          server-addr: 127.0.0.1:8848

```

