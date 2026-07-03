# Design a Rate Limiter 复习笔记

这组笔记整理自截图内容，主题是设计 API Rate Limiter。笔记按面试回答流程拆分：先确认问题范围和需求，再给出高层设计，然后深入限流算法，最后补充规则配置和分布式实现细节。

## 笔记目录

1. [问题背景、收益与需求](01-problem-scope-and-requirements.md)
2. [限流器放在哪里与高层设计](02-placement-and-high-level-design.md)
3. [限流算法详解](03-rate-limiting-algorithms.md)
4. [规则配置、深挖设计与面试总结](04-rules-deep-dive-and-summary.md)

## 总体思路

```mermaid
flowchart TD
    A["1. Understand scope<br/>What kind of limiter? server-side API limiter"]
    B["2. Requirements<br/>Accurate, low latency, memory efficient, distributed, fault tolerant"]
    C["3. Placement<br/>Server-side middleware or API gateway"]
    D["4. Algorithm choice<br/>Token bucket / leaky bucket / fixed window / sliding window"]
    E["5. Storage<br/>Counters or buckets in Redis / distributed cache"]
    F["6. Decision<br/>Allow request or return HTTP 429"]
    G["7. Rules<br/>Declarative rate limit configs"]
    H["8. Monitoring<br/>Track rejects, latency, quota usage"]

    A --> B --> C --> D --> E --> F --> G --> H
```

## 一句话总览

Rate limiter 的核心是控制某个主体在某段时间内能发多少请求。设计时要明确限流对象、限流维度、限流算法、限流状态存储、分布式一致性、异常处理和被限流后的响应方式。
