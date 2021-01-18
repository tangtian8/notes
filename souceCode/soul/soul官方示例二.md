## Dubbo服务接入soul网关

1. 支持 alibaba dubbo（< 2.7.x） 以及 apache dubbo (>=2.7.x)。
2. 官方提供的各种demo https://github.com/dromara/soul/tree/master/soul-examples

```
.
├── pom.xml
├── soul-examples-dubbo
├── soul-examples-eureka
├── soul-examples-http
├── soul-examples-sofa
├── soul-examples-springcloud
├── soul-examples-tars
└── soul-examples.iml
```

### soul-examples-dubbo项目结构

```
.
├── pom.xml
├── soul-examples-alibaba-dubbo-service
├── soul-examples-apache-dubbo-service
├── soul-examples-dubbo-api
```

### soul-examples-apache-dubbo-service项目结构

```
.
├── pom.xml
├── soul-examples-apache-dubbo-service.iml
├── src
│   └── main
│       ├── java
│       │   └── org
│       │       └── dromara
│       │           └── soul
│       │               └── examples
│       │                   └── apache
│       │                       └── dubbo
│       │                           └── service
│       │                               ├── TestApacheDubboApplication.java
│       │                               └── impl
│       │                                   ├── DubboMultiParamServiceImpl.java
│       │                                   └── DubboTestServiceImpl.java
│       └── resources
│           ├── application.yml
│           └── spring-dubbo.xml
```

引入的maven依赖

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-apache-dubbo</artifactId>
            <version>2.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.5</version>
        </dependency>
```

配置文件application.yml

```
server:
  port: 8011
  address: 0.0.0.0
  servlet:
    context-path: /
spring:
  main:
    allow-bean-definition-overriding: true
soul:
  dubbo:
    adminUrl: http://localhost:9095
    contextPath: /dubbo
    appName: dubbo
```

配置文件spring-dubbo.xml

修改自己的zookeeper地址

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <dubbo:application name="test-dubbo-service"/>

    <dubbo:registry address="zookeeper://120.79.218.62:2181"/>

    <dubbo:protocol name="dubbo" port="20888"/>

    <dubbo:service timeout="10000" interface="org.dromara.soul.examples.dubbo.api.service.DubboTestService" ref="dubboTestService"/>
    <dubbo:service timeout="10000" interface="org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService" ref="dubboMultiParamService"/>
</beans>
```

需要在 DubboTestService 的 **方法上**，添加 [`@SoulDubboClient`](https://github.com/Dromara/soul/blob/master/soul-client/soul-client-http/soul-client-springmvc/src/main/java/org/dromara/soul/client/springmvc/annotation/SoulSpringMvcClient.java)

```
@Service("dubboTestService")
public class DubboTestServiceImpl implements DubboTestService {
    @Override
    @SoulDubboClient(path = "/findById", desc = "Query by Id")
    public DubboTest findById(final String id) {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setId(id);
        dubboTest.setName("hello world Soul Apache, findById");
        return dubboTest;
    }
}
```

### 启动项目

1. 启动soul-admin
2. 启动soul-bootstrap 网关地址: http://localhost:9195/
3. 启动soul.examples.apache.dubbo.service [http://localhost:](http://localhost:9195/)8011[/](http://localhost:9195/)
4. 进入soul-admin http://localhost:9095/ 用户名：admin 密码：123456
5. 可看见dubbo插件

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610767389930-64c42033-ff82-4356-a511-6c76f5c892ba.png?x-oss-process=image%2Fresize%2Cw_1500)

## Dubbo服务接入soul网关功能演示

进入soul-admin

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610767693775-5b7e90c0-563c-4159-a3b3-0b6769081b0d.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610767740903-bdeef84b-cd7d-4c44-9904-bd79e67d11b9.png)

##### 通过网关的请求接口

dobbo服务

```
    @Override
    @SoulDubboClient(path = "/findById", desc = "Query by Id")
    public DubboTest findById(final String id) {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setId(id);
        dubboTest.setName("hello world Soul Apache, findById");
        return dubboTest;
    }
```

请求地址

http://localhost:9195/dubbo/findById?id=1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610769012560-58f26eb8-5529-4600-aadb-4eb89430a32a.png)