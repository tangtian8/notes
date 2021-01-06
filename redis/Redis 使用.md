# Redis 使用

Redis一般有以下应用:

String：缓存、限流、计数器、分布式锁、分布式Session

Hash：存储用户信息、用户主页访问量、组合查询

List：微博关注人时间轴列表、简单队列

Set：赞、踩、标签、好友关系

Zset：排行榜

## [Redis命令](http://www.redis.cn/commands.html) 

Redis命令十分丰富，包括的命令组有Cluster、Connection、Geo、Hashes、HyperLogLog、Keys、Lists、Pub/Sub、Scripting、Server、Sets、Sorted Sets、Strings、Transactions一共14个redis命令组两百多个redis命令。

cmd访问redis

```
redis-cli -h 127.0.0.1 -p 6379
```

key


```
    keys * 获取所有的key
    select 0 选择第一个库
    move myString 1 将当前的数据库key移动到某个数据库,目标库有，则不能移动
    flushdb      清除指定库
    randomkey     随机key
    type key      类型
    
    set key1 value1 设置key
    get key1    获取key
    mset key1 value1 key2 value2 key3 value3
    mget key1 key2 key3
    del key1   删除key
    exists key      判断是否存在key
    expire key 10   10过期
    pexpire key 1000 毫秒
    persist key     删除过期时间
```

string

```
    set name cxx
    get name
    getrange name 0 -1        字符串分段
    getset name new_cxx       设置值，返回旧值
    mset key1 key2            批量设置
    mget key1 key2            批量获取
    setnx key value           不存在就插入（not exists）
    setex key time value      过期时间（expire）
    setrange key index value  从index开始替换value
    incr age        递增
    incrby age 10   递增
    decr age        递减
    decrby age 10   递减
    incrbyfloat     增减浮点数
    append          追加
    strlen          长度
    getbit/setbit/bitcount/bitop    位操作
```

hash

```
    hset myhash name cxx
    hget myhash name
    hmset myhash name cxx age 25 note "i am notes"
    hmget myhash name age note   
    hgetall myhash               获取所有的
    hexists myhash name          是否存在
    hsetnx myhash score 100      设置不存在的
    hincrby myhash id 1          递增
    hdel myhash name             删除
    hkeys myhash                 只取key
    hvals myhash                 只取value
    hlen myhash                  长度
```

list

```
    lpush mylist a b c  左插入
    rpush mylist x y z  右插入
    lrange mylist 0 -1  数据集合
    lpop mylist  弹出元素
    rpop mylist  弹出元素
    llen mylist  长度
    lrem mylist count value  删除
    lindex mylist 2          指定索引的值
    lset mylist 2 n          索引设值
    ltrim mylist 0 4         删除key
    linsert mylist before a  插入
    linsert mylist after a   插入
    rpoplpush list list2     转移列表的数据
```

Set

```
    sadd myset redis 
    smembers myset       数据集合
    srem myset set1         删除
    sismember myset set1 判断元素是否在集合中
    scard key_name       个数
    sdiff | sinter | sunion 操作：集合间运算：差集 | 交集 | 并集
    srandmember          随机获取集合中的元素
    spop                 从集合中弹出一个元素
```

Zset

```
    zadd zset 1 one
    zadd zset 2 two
    zadd zset 3 three
    zincrby zset 1 one              增长分数
    zscore zset two                 获取分数
    zrange zset 0 -1 withscores     范围值
    zrangebyscore zset 10 25 withscores 指定范围的值
    zrangebyscore zset 10 25 withscores limit 1 2 分页
    Zrevrangebyscore zset 10 25 withscores  指定范围的值
    zcard zset  元素数量
    Zcount zset 获得指定分数范围内的元素个数
    Zrem zset one two        删除一个或多个元素
    Zremrangebyrank zset 0 1  按照排名范围删除元素
    Zremrangebyscore zset 0 1 按照分数范围删除元素
    Zrank zset 0 -1    分数最小的元素排名为0
    Zrevrank zset 0 -1  分数最大的元素排名为0
    Zinterstore
    zunionstore rank:last_week 7 rank:20150323 rank:20150324 rank:20150325  weights 1 1 1 1 1 1 1
```



排序

```
    sort mylist  排序
    sort mylist alpha desc limit 0 2 字母排序
    sort list by it:* desc           by命令
    sort list by it:* desc get it:*  get参数
    sort list by it:* desc get it:* store sorc:result  sort命令之store参数：表示把sort查询的结果集保存起来
```

订阅与发布

```
    订阅频道：subscribe chat1
    发布消息：publish chat1 "hell0 ni hao"
    查看频道：pubsub channels
    查看某个频道的订阅者数量: pubsub numsub chat1
    退订指定频道： unsubscrible chat1   , punsubscribe java.*
    订阅一组频道： psubscribe java.*
```

Redis 事务

```
     隔离性，原子性， 
     步骤：  开始事务，执行命令，提交事务
             multi  //开启事务
             sadd myset a b c
             sadd myset e f g
             lpush mylist aa bb cc
             lpush mylist dd ff gg
```

服务器管理

```
    dump.rdb
    appendonly.aof
    //BgRewriteAof 异步执行一个aop(appendOnly file)文件重写
    会创建当前一个AOF文件体积的优化版本
    
    //BgSave 后台异步保存数据到磁盘，会在当前目录下创建文件dump.rdb
    //save同步保存数据到磁盘，会阻塞主进程，别的客户端无法连接
    
    //client kill 关闭客户端连接
    //client list 列出所有的客户端
    
    //给客户端设置一个名称
      client setname myclient1
      client getname
      
     config get port
     //configRewrite 对redis的配置文件进行改写
```

rdb

```
save 900 1
save 300 10
save 60 10000
```

aop备份处理

```
appendonly yes 开启持久化
appendfsync everysec 每秒备份一次
```

命令：

```
bgsave异步保存数据到磁盘（快照保存）
lastsave返回上次成功保存到磁盘的unix的时间戳
shutdown同步保存到服务器并关闭redis服务器
bgrewriteaof文件压缩处理（命令）
```

## [管道（Pipelining）](http://www.redis.cn/topics/pipelining.html)

### 请求/响应协议和RTT

Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。

这意味着通常情况下一个请求会遵循以下步骤：

- 客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
- 服务端处理命令，并将结果返回给客户端。

无论网络延如何延时，数据包总是能从客户端到达服务器，并从服务器返回数据回复客户端。

### Redis 管道（Pipelining）

一次请求/响应服务器能实现处理新的请求即使旧的请求还未被响应。这样就可以将*多个命令*发送到服务器，而不用等待回复，最后在一个步骤中读取该答复。

### 管道（Pipelining） VS 脚本（Scripting）

大量 pipeline 应用场景可通过 Redis [脚本](http://www.redis.cn/commands/eval.html)（Redis 版本 >= 2.6）得到更高效的处理，后者在服务器端执行大量工作。脚本的一大优势是可通过最小的延迟读写数据，让读、计算、写等操作变得非常快（pipeline 在这种情况下不能使用，因为客户端在写命令前需要读命令返回的结果）。

应用程序有时可能在 pipeline 中发送 [EVAL](http://www.redis.cn/commands/eval.html) 或 [EVALSHA](http://www.redis.cn/commands/evalsha.html) 命令。Redis 通过 [SCRIPT LOAD](http://www.redis.cn/commands/script-load.html) 命令（保证 EVALSHA 成功被调用）明确支持这种情况。

## [Redis 发布/订阅（Pub/Sub）](http://www.redis.cn/topics/pubsub.html)

为了订阅foo和bar，客户端发出一个订阅的频道名称

```
SUBSCRIBE foo bar
```

从另一个客户端我们发出关于频道名称为second的发布操作:

```
 PUBLISH foo Hello
```

## [Redis Lua 脚本](http://www.redis.cn/commands/eval.html)

### EVAL script numkeys key [key ...] arg [arg ...]

EVAL 和 EVALSHA 命令是从 Redis 2.6.0 版本开始的，使用内置的 Lua 解释器，可以对 Lua 脚本进行求值。

EVAL的第一个参数是一段 Lua 5.1 脚本程序。 这段Lua脚本不需要（也不应该）定义函数。它运行在 Redis 服务器中。

EVAL的第二个参数是参数的个数，后面的参数（从第三个参数），表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 KEYS 数组，用 1 为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。

在命令的最后，那些不是键名参数的附加参数 arg [arg …] ，可以在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。

```
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

这是从一个Lua脚本中使用两个不同的Lua函数来调用Redis的命令的例子

```
redis.call()
redis.pcall()
```

redis.call() 与 redis.pcall()很类似, 他们唯一的区别是当redis命令执行结果返回错误时， redis.call()将返回给调用者一个错误，而redis.pcall()会将捕获的错误以Lua表的形式返回

redis.call() 和 redis.pcall() 两个函数的参数可以是任意的 Redis 命令：