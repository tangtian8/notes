## Divide插件介绍

据官方文档介绍，divide插件是网关处理 `http协议`请求的核心处理插件。

## Divide插件的父类

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611533797510-017b8778-7c39-478a-ae35-ed5414a9465e.png)

插件在AbstractSoulPlugin 类中通过execute方法进行Web预处理。

通过插件名字获取插件信息

```
 BaseDataCache.getInstance().obtainPluginData(pluginName);
```

筛选相关插件，被开启的插件才可以用

```
pluginData != null && pluginData.getEnabled()
```

查找选择器数据

```
 BaseDataCache.getInstance().obtainSelectorData(pluginName);
```

匹配选择器

```
matchSelector(exchange, selectors)
```

获取规则数据，规则根据选择器类型来定。如果自定义规则我们就要做一下匹配matchRule(exchange, rules);

```
BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
```

上述数据都不为空我们就执行doExecute(exchange, chain, selectorData, rule);

## Divide插件的doExecute方法

doExecute方法重写了父类的doExecute方法



在ServerWebExchange类中拿到SoulContext的实例

```
exchange.getAttribute(Constants.CONTEXT);
```

获取上游数据列表

```
 UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
```

获取divideUpstream 上游数据

```
 LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
```

如果上述数据为null 或者为空就抛异常

如果正确就构建http url

```
String domain = buildDomain(divideUpstream);
String realURL = buildRealURL(domain, soulContext, exchange);
```

并且设置http属性

```
exchange.getAttributes().put(Constants.HTTP_URL, realURL);
exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
```

再次处理后续插件

chain.execute(exchange);



## 总结

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2542319/1611537603431-9c5bfae0-2f5a-466c-8059-06b6504deb17.png)

通过打印插件顺序我们可知，

最开始执行了全局插件，如waf，rate_limiter，hystrix插件

然后是自定义插件divide

通过WebClientPlugin插件请求接口

最后通过WebClientResponsePlugin 插件返回数据