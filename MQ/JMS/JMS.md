# JMS规范
https://www.cnblogs.com/leeSmall/p/9638248.html
## 1. 什么是JMS规范

JMS（Java Messaging Service）规范，本质是API，Java平台消息中间件的规范，java应用程序之间进行消息交换。并且通过提供标准的产生、发送、接收消息的接口简化企业应用的开发。对应的实现ActiveMQ

## 2. JMS对象模型包含如下几个要素

1）连接工厂：创建一个JMS连接
2）JMS连接：客户端和服务器之间的一个连接。
3）JMS会话：客户端和服务器会话的状态，建立在连接之上的
4）JMS目的：消息队列
5）JMS生产者：消息的生成
6）JMS消费者：接收和消费消息
7）Broker：消息中间件的实例（ActiveMQ）

## 3. JMS规范中的点对点模式

![img](https://images2018.cnblogs.com/blog/1227483/201809/1227483-20180913003113549-929824957.png)

**队列，一个消息只有一个消费者（即使有多个接受者监听队列），消费者是要向队列应答成功**

## 4. JMS规范中的主题模式（发布订阅）

![img](https://images2018.cnblogs.com/blog/1227483/201809/1227483-20180913003025104-1588623564.png)

**发布到Topic的消息会被当前主题所有的订阅者消费**

## 5. JMS规范中的消息类型

TextMessage,MapMessage,ObjectMessage,BytesMessage,StreamMessage