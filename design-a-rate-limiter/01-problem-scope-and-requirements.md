# 01. 问题背景、收益与需求

## 1. Rate Limiter 是什么

Rate limiter 用来控制客户端或服务发送流量的速率。在 HTTP API 场景中，它限制客户端在指定时间窗口内允许发送的请求数量。

可以用奶茶店来理解：

- 奶茶店每分钟最多做 10 杯奶茶。
- 如果突然 100 个人同时下单，店员、机器和排队系统都会崩。
- 所以店里可以定规则：每个用户每分钟最多下 2 单，整个系统每分钟最多接 10 单。
- 超过的订单要么拒绝，要么排队。

系统里的 rate limiter 也是这个意思：防止某个用户、IP、设备或 API key 在短时间内发太多请求，把后端服务、数据库或第三方 API 打爆。

如果请求数量超过阈值，额外请求会被阻止，通常返回：

```text
HTTP 429 Too Many Requests
```

## 2. 常见限流例子

截图中给了几个例子：

- 一个用户每秒最多写 2 篇 posts。
- 同一个 IP 地址每天最多创建 10 个 accounts。
- 同一个 device 每周最多领取 5 次 rewards。

这些例子说明 rate limiting 不一定只按用户限流，也可以按 IP、device、API key、账号、endpoint 等维度限流。

更贴近系统设计的规则例子：

```text
登录接口：每个用户每分钟最多 5 次
发短信接口：每个手机号每天最多 10 次
发帖接口：每个用户每秒最多 2 次
全站接口：整个系统每秒最多 10000 次
```

规则通常由两部分组成：

- 按谁限流：`userId`、`IP`、`deviceId`、手机号、API key、endpoint。
- 限多少：每秒、每分钟、每天最多多少次。

例如登录防刷：

```text
key = login:user_123
limit = 5 次 / 分钟
```

例如短信防刷：

```text
key = sms:phone_138xxxx
limit = 10 次 / 天
```

## 3. 为什么需要 Rate Limiter

### 3.1 防止资源耗尽和 DoS

Rate limiter 可以阻止恶意或异常流量耗尽系统资源。

如果没有限流：

- 某个用户或攻击者可能疯狂调用 API。
- 服务器、数据库、缓存、第三方 API 都可能被打爆。
- 正常用户请求被拖慢或失败。

Rate limiter 能阻止 intentional 或 unintentional 的过量请求。

### 3.2 降低成本

限制过量请求意味着：

- 需要的服务器更少。
- 数据库压力更低。
- 对付费第三方 API 的调用更少。

对按调用次数计费的 API 尤其重要，比如：

- credit check
- payment
- health record retrieval
- maps / geocoding API

### 3.3 防止服务器过载

Rate limiter 可以过滤 bot 或用户误操作产生的 excess requests，避免服务器被突发请求压垮。

常见场景：

- bot 爬虫。
- 用户疯狂刷新。
- 客户端 bug 导致无限重试。
- 下游服务变慢后，上游不断重试。

## 4. Step 1：理解问题并确定范围

截图中面试对话强调：Rate limiting 可以用不同算法实现，每种有优缺点。候选人和面试官要先澄清正在设计哪类限流器。

常见澄清问题：

### 4.1 限流器类型

问题：我们设计哪种 rate limiter？

这里聚焦：

- Server-side API rate limiter。

也就是说，限流逻辑在服务端或网关侧，而不是客户端自己限制。

### 4.2 限流依据

问题：是否基于 IP、user ID 或其他属性限流？

答案：Rate limiter 应该足够灵活，支持多种 throttle rules。

可能的限流维度：

- user ID
- IP address
- API key
- device ID
- endpoint
- tenant / organization
- region

### 4.3 系统规模

问题：系统是 startup 级别还是大用户规模？

答案：系统要能处理大量请求。

这意味着：

- 限流判断必须低延迟。
- 存储必须高吞吐。
- 多实例之间要共享限流状态。

### 4.4 分布式环境

问题：系统是否运行在分布式环境？

答案：是。

这要求：

- 多台 API server 看到一致或近似一致的限流状态。
- 不能只把 counter 存在某一台 server 本地内存里。

### 4.5 是否独立服务

问题：Rate limiter 是独立服务，还是集成在应用代码中？

答案：这是设计决策。

可以选择：

- 作为独立 service。
- 作为 middleware。
- 放在 API gateway。
- 集成在应用代码里。

### 4.6 是否通知用户被限流

问题：是否需要通知用户请求被 throttled？

答案：是。

通常返回：

- HTTP 429 Too Many Requests。
- 响应头中带剩余额度和重试时间。

## 5. 系统需求总结

截图中的 requirements 包括：

- Accurately limit excessive requests。
- Low latency。
- Use as little memory as possible。
- Distributed rate limiting。
- Exception handling。
- High fault tolerance。

逐条解释如下。

### 5.1 准确限流

系统应该准确限制过量请求，不能让明显超额的请求继续通过。

但注意：在大规模分布式系统里，绝对精确可能很贵，很多系统接受近似精确。

### 5.2 低延迟

Rate limiter 位于请求路径上，不能显著拖慢 HTTP response time。

如果每个请求都要经过限流器，限流器本身必须非常快。

### 5.3 内存效率

Rate limiter 需要保存每个限流对象的状态，例如 counter、bucket tokens、timestamps。

用户量大时，状态数量巨大，所以要尽量节省内存。

### 5.4 分布式限流

Rate limiter 需要跨多个 servers / processes 工作。

如果每台服务器只在本地计数，同一个用户的请求打到不同 server 时就可能绕过限制。

### 5.5 异常处理

当用户请求被 throttle 时，需要明确告知用户。

常见方式：

```text
HTTP 429 Too Many Requests
Retry-After: 10
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1720000000
```

### 5.6 高容错

Rate limiter 出问题时，不应该影响整个系统。

需要考虑：

- 限流存储不可用怎么办？
- 限流服务超时怎么办？
- 是 fail open 还是 fail closed？

## 6. 面试复习点

- Rate limiter 用来控制单位时间内的请求数量。
- 限流维度可以是 user ID、IP、API key、device ID、endpoint 等。
- 返回 429 是最常见的被限流响应。
- Rate limiter 的核心需求是准确、低延迟、节省内存、分布式、异常处理和高容错。
- 面试第一步不要急着讲算法，先确认限流范围和业务规则。
- 限流规则本质上就是：按哪个 key 统计，在多长时间内最多允许多少次。
