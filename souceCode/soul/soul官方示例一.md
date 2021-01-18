## 网关对http的代理

1. soul网关使用 divide 插件来处理http请求
2. [divide 插件](https://dromara.org/zh-cn/docs/soul/plugin-divide.html)实现 HTTP **正向**代理的功能，所有 HTTP 类型的请求，都由它进行**负载均衡**的调用。
3. 官方提供的各种demo https://github.com/dromara/soul/tree/master/soul-examples

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



### soul-examples-http项目结构

soul-examples-http

```
.
├── pom.xml
├── soul-examples-http.iml
├── src
│   └── main
│       ├── java
│       │   └── org
│       │       └── dromara
│       │           └── soul
│       │               └── examples
│       │                   └── http
│       │                       ├── SoulTestHttpApplication.java
│       │                       ├── config
│       │                       │   ├── HttpServerConfig.java
│       │                       │   └── WebSocketConfiguration.java
│       │                       ├── controller
│       │                       │   ├── HttpTestController.java
│       │                       │   └── OrderController.java
│       │                       ├── dto
│       │                       │   ├── OrderDTO.java
│       │                       │   ├── RequestDTO.java
│       │                       │   └── UserDTO.java
│       │                       ├── result
│       │                       │   ├── PaymentTypeService.java
│       │                       │   └── ResultBean.java
│       │                       ├── router
│       │                       │   └── SoulTestHttpRouter.java
│       │                       └── websocket
│       │                           └── EchoHandler.java
│       └── resources
│           └── application.yml
```

引入的maven依赖

```
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
  <version>2.2.1</version>
</dependency>
```

配置文件

```
soul:
  http:
    adminUrl: http:localhost:9095 #Soul Admin 地址
    port: 8188 #本项目的启动端口
    contextPath: /http #设置在 Soul 网关的路由前缀，例如说 /order、/product
    appName: http #应用名。未配置情况下，默认使用 `spring.application.name` 配置项
    full: false #设置true 代表代理你的整个服务，false表示代理你其中某几个controller
```

需要在 Controller 的 **HTTP API 方法上**，添加 [`@SoulSpringMvcClient`](https://github.com/Dromara/soul/blob/master/soul-client/soul-client-http/soul-client-springmvc/src/main/java/org/dromara/soul/client/springmvc/annotation/SoulSpringMvcClient.java)

```
@RestController
@RequestMapping("/test")
@SoulSpringMvcClient(path = "/test/**")
public class HttpTestController {

    /**
     * Post user dto.
     *
     * @param userDTO the user dto
     * @return the user dto
     */
    @PostMapping("/payment")
    public UserDTO post(@RequestBody final UserDTO userDTO) {
        return userDTO;
    }

    /**
     * Find by user id string.
     *
     * @param userId the user id
     * @return the string
     */
    @GetMapping("/findByUserId")
    public UserDTO findByUserId(@RequestParam("userId") final String userId) {
        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(userId);
        userDTO.setUserName("hello world");
        return userDTO;
    }
 }
```

### 启动项目

1. 启动soul-admin
2. 启动soul-bootstrap 网关地址: http://localhost:9195/
3. 启动soul-examples-http [http://localhost:](http://localhost:9195/)8188[/](http://localhost:9195/)
4. 进入soul-admin http://localhost:9095/ 用户名：admin 密码：123456
5. 可看见divide插件已开启

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610707983548-d5da0242-9be2-49be-9ffd-129f1eaddc16.png?x-oss-process=image%2Fresize%2Cw_1500)

## http网关代理功能演示

#### 转发功能

| 服务名称           | 服务端口                       | 功能说明   |
| ------------------ | ------------------------------ | ---------- |
| soul-admin         | [9095](http://localhost:9095/) | 管理控制台 |
| soul-examples-http | 8188                           | 正式的项目 |
| soul-bootstrap     | 9195                           | 网关地址   |

##### 转发流程

![img](https://cdn.nlark.com/yuque/__puml/98cfb7ab7e7b22ac4b91d36a0aa813d9.svg)

##### 真实请求接口

http://localhost:8188/order/findById?id=1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610710428759-24511e8e-e905-411a-99e6-fcd383add05e.png?x-oss-process=image%2Fresize%2Cw_1500)

##### 通过网关的请求接口

网关地址的定义规则

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610710880749-b1219c32-8144-4fa8-9730-97045b1c652c.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610710991603-047442f1-1404-4d4d-8a28-8e1aa2b1644b.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610710655472-d94eb43b-88ab-4037-8b12-5e4f9e56b2a1.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610710624467-964f845f-647e-4cac-8a03-8d4fbe05ee12.png)

通过网关访问

http://localhost:9195/http/order/findById?id=1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711013700-48a39663-6391-4f11-8dd8-2d9a15b6f7a3.png?x-oss-process=image%2Fresize%2Cw_1500)

和真实的服务地址访问结果比较一致

#### 负载均衡功能

| 服务名称           | 服务端口                       | 功能说明   |
| ------------------ | ------------------------------ | ---------- |
| soul-admin         | [9095](http://localhost:9095/) | 管理控制台 |
| soul-examples-http | 8188                           | 正式的项目 |
| soul-examples-http | 8189                           | 正式的项目 |
| soul-bootstrap     | 9195                           | 网关地址   |

##### 再次启动一个8189的soul-examples-http

更改并行启动

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711330718-158605ab-bf3d-432f-9775-5f8ad5823fcf.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711359549-fe47acdc-807c-47bd-85a9-73649e10f96f.png?x-oss-process=image%2Fresize%2Cw_1500)

更改端口

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711517668-493add89-1bfb-4a3f-89ab-f067150451eb.png?x-oss-process=image%2Fresize%2Cw_1500)

启动

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711538816-31406a89-34a8-4f2e-823a-6a8209eec35f.png?x-oss-process=image%2Fresize%2Cw_1500)

进入控制台

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711639194-1d7ee58a-c84e-48f6-9600-5fd9d9e488ad.png)

##### 权重配置

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711763773-510dbf38-8f8f-4c85-925f-6327d5baa007.png)

weight为100的port是8189 、weight为1的port是8188

##### 验证结果

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610711966661-e9ad169d-98f7-411a-9650-d374a579d57d.png)

结果基本上都在8189的链接上



#### 对每个url的规则修改

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1610712136141-fce3f241-acff-4fe0-8ebd-7e05f5201b5e.png)

### 如何做请求转发？

### 如何做负载均衡？