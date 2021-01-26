## 环境准备

Soul-Admin的yml文件中

```
soul:
  sync:
      http:
        enabled: true
```



Soul-Bootstrap项目的pom文件引入

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
            <version>${project.version}</version>
        </dependency>
```

Soul-Bootstrap项目的yml文件文件引入

```
soul :
  sync:
     http:
           url : http://localhost:9095
```

## soulAdmin长轮询分析

获取配置文件的请求的接口，目前Soul-Bootstrap端用于获取配置信息。

```
    @GetMapping("/fetch")
    public SoulAdminResult fetchConfigs(@NotNull final String[] groupKeys) {
       
    }
```

保持长链接的接口，目前Soul-Bootstrap端用于数据交互

```
    @PostMapping(value = "/listener")
    public void listener(final HttpServletRequest request, final HttpServletResponse response) {
        longPollingListener.doLongPolling(request, response);
    }
```

研究一下doLongPolling(request, response)方法的实现

开启了一个定时任务，处理实例化的LongPollingClient

```
scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
```

LongPollingClient放在一个BlockingQueue的队列里面

```
  private final BlockingQueue<LongPollingClient> clients;
```

LongPollingClient自身如果超 60S且仍没有数据更改，则返回空数据。如果数据在此时间范围内发生更改，则取消的定时任务并响应更改后的组数据。

```
 class LongPollingClient implements Runnable {}
```

DataChangeTask当group的数据更改时，将创建线程以异步通知客户端

```
 class DataChangeTask implements Runnable {}
```

## soulBootstrap的长轮询分析

HttpSyncDataService 的构造方法中，new这个对象时，会调用这个方法。

```
this.start();
```

当有多个线程同时执行这些类的实例包含的方法时，具有排他性，即当某个线程进入方法，执行其中的指令时，不会被其他线程打断，而别的线程就像自旋锁一样，一直等到该方法执行完成

```
private static final AtomicBoolean RUNNING = new AtomicBoolean(false);
RUNNING.compareAndSet(false, true)
```

通过httpClient工具请求soulAdmin获取所有配置信息

```
this.fetchGroupConfig(ConfigGroupEnum.values());
```

创建线程池，开启长连接，用于更新。

```
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // start long polling, each server creates a thread to listen for changes.
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
```

用于跟Soul-Admin通信的长轮询方法

```
    @SuppressWarnings("unchecked")
    private void doLongPolling(final String server) {
        ···
        String listenerUrl = server + "/configs/listener";
        ···
        String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
        ···
    }
```