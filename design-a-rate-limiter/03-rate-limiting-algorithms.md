# 03. 限流算法详解

## 1. 常见限流算法

截图中列出常见算法：

- Token bucket
- Leaking bucket
- Fixed window counter
- Sliding window log
- Sliding window counter

每个算法都有优缺点，面试中不一定要全部实现，但要能解释适用场景和 trade-off。

## 2. Token Bucket Algorithm

Token bucket 是最常用的限流算法之一，很多互联网公司使用它，例如 Amazon 和 Stripe。

核心思想：

- 有一个 bucket，容量固定。
- 系统按照固定速率往 bucket 里放 tokens。
- 每个请求到来时消耗一个 token。
- 如果 bucket 里有 token，请求通过。
- 如果没有 token，请求被拒绝。
- bucket 满了以后，多余 token 会溢出，不再增加。

## 3. 图 4-4：Token Bucket 基本模型

```mermaid
flowchart TD
    Refill["Refill tokens periodically"] --> Bucket[("Token Bucket<br/>capacity = N")]
    Req["Incoming request"] --> Check{"Enough tokens?"}
    Bucket --> Check
    Check -- "Yes: consume 1 token" --> Pass["Request goes through"]
    Check -- "No tokens" --> Drop["Request is dropped"]
```

## 4. 图 4-5：Token Bucket 请求处理流程

```mermaid
flowchart LR
    Requests["Requests"] --> Check{"Bucket has tokens?"}
    Refill["Refiller"] --> Bucket[("Token Bucket")]
    Bucket --> Check
    Check -- "Yes, consume token" --> Forward["Forward requests"]
    Check -- "No" --> Drop["Drop requests"]
```

## 5. 图 4-6：Token Bucket 示例

截图中例子：

- Bucket size = 4。
- Refill rate = 4 tokens / minute。

过程：

1. 开始有 4 个 tokens，请求通过并消耗 tokens。
2. 如果短时间内来了 3 个请求，消耗 3 个 tokens。
3. 如果 bucket 里没有 token，请求被拒绝。
4. 每分钟补充 4 个 tokens。

```mermaid
sequenceDiagram
    participant R as Requests
    participant B as Bucket size 4
    participant API as API

    Note over B: Start with 4 tokens
    R->>B: 3 requests
    B-->>API: 3 pass, consume 3 tokens
    Note over B: 1 token left
    R->>B: 2 more requests
    B-->>API: 1 pass
    B-->>R: 1 dropped
    Note over B: Refill 4 tokens per minute, capped at 4
```

## 6. Token Bucket 参数

Token bucket 有两个关键参数：

### 6.1 Bucket Size

Bucket size 是 bucket 能容纳的最大 token 数量。

它决定允许多少短时间 burst。

例如：

- bucket size = 4，最多可以瞬间通过 4 个请求。
- bucket size = 100，允许更大的突发流量。

### 6.2 Refill Rate

Refill rate 是系统每秒或每分钟放入 bucket 的 token 数量。

它决定长期平均速率。

例如：

- 4 tokens / minute。
- 100 tokens / second。

## 7. 需要多少个 Bucket

Bucket 数量取决于限流规则。

例子：

- 如果按 user 限流，每个 user 一个 bucket。
- 如果按 IP 限流，每个 IP 一个 bucket。
- 如果按 endpoint 限流，每个 endpoint 一个 bucket。
- 如果一个用户对多个 API 有不同限额，可以为不同 API 建不同 bucket。

截图中例子：

- 用户每天最多发 1 post。
- 用户每天最多加 150 friends。
- 每个用户需要 2 个 buckets。

## 8. Token Bucket 优缺点

优点：

- 算法简单。
- 内存效率较好。
- 允许短时间 burst。
- 适合大多数 API 限流场景。

缺点：

- 需要调两个参数：bucket size 和 refill rate。
- 参数设置不好会造成过松或过严。

## 9. Leaky Bucket Algorithm

Leaky bucket 和 token bucket 类似，但工作方式不同。

核心思想：

- 请求进入一个 queue。
- queue 以固定速率流出请求。
- 如果 queue 满了，新请求被丢弃。

它像一个漏水的桶：水可以突发进入桶，但流出速度固定。

## 10. 图 4-7：Leaky Bucket 工作流程

```mermaid
flowchart LR
    Req["Requests"] --> Full{"Bucket / Queue full?"}
    Full -- "Yes" --> Drop["Drop requests"]
    Full -- "No" --> Queue["Queue"]
    Queue -- "Processed at fixed rate" --> Worker["Requests processed"]
```

## 11. Leaky Bucket 参数

Leaky bucket 有两个关键参数：

### 11.1 Bucket Size

队列大小，决定最多可以排队多少请求。

### 11.2 Outflow Rate

固定处理速率，决定每秒或每分钟处理多少请求。

## 12. Leaky Bucket 优缺点

优点：

- 内存效率高，因为 queue 大小固定。
- 请求按固定速率处理。
- 适合需要稳定输出速率的场景。

缺点：

- 突发流量会填满队列，旧请求如果不能及时处理，新的请求会被限流。
- 参数有两个，调参不容易。
- 可能增加请求等待时间。

## 13. Fixed Window Counter Algorithm

Fixed window counter 把时间切成固定窗口，每个窗口维护一个 counter。

规则：

- 每个窗口开始时 counter 重置为 0。
- 每个请求到来时 counter +1。
- 如果 counter 小于等于阈值，请求通过。
- 如果 counter 超过阈值，请求被拒绝。

## 14. 图 4-8 / 4-9：Fixed Window 示例

例子：

- 每分钟最多 3 个请求。
- 每个分钟窗口独立计数。
- 超过 3 个的请求被拒绝。

```mermaid
flowchart LR
    W1["1:00:00 - 1:00:59<br/>allowed up to 3"]
    W2["1:01:00 - 1:01:59<br/>allowed up to 3"]
    W3["1:02:00 - 1:02:59<br/>allowed up to 3"]

    W1 --> W2 --> W3
```

固定窗口的问题在边界处最明显：

- 1:00:58 到 1:00:59 通过 3 个请求。
- 1:01:00 到 1:01:01 又通过 3 个请求。
- 2 秒内实际通过了 6 个请求。

这会形成 burst。

## 15. Fixed Window 优缺点

优点：

- 内存效率高。
- 容易理解。
- 实现简单。
- 适合某些粗粒度限流。

缺点：

- 窗口边界可能出现流量突刺。
- 在窗口边缘通过的请求可能超过期望速率。

## 16. Sliding Window Log Algorithm

Sliding window log 记录每个请求的 timestamp。

流程：

1. 新请求到达。
2. 删除所有已经过期的 timestamps。
3. 统计当前窗口内的 timestamps 数量。
4. 如果数量小于限制，请求通过，并把当前 timestamp 加入 log。
5. 如果数量达到限制，请求被拒绝。

## 17. 图 4-10：Sliding Window Log 示例

假设允许每分钟 2 个请求。

```mermaid
sequenceDiagram
    participant R as Request
    participant Log as Timestamp Log
    participant API as API

    R->>Log: New request at 1:00:01
    Log->>Log: Remove timestamps older than 1 minute
    Log->>Log: Count timestamps in window
    alt Count < limit
        Log->>Log: Add timestamp
        Log-->>API: Allow
    else Count >= limit
        Log-->>R: Reject
    end
```

## 18. Sliding Window Log 优缺点

优点：

- 非常准确。
- 没有 fixed window 的边界突刺问题。

缺点：

- 内存消耗大。
- 即使请求被拒绝，timestamp 可能仍需要存储或检查。
- 高流量场景下日志量很大。

## 19. Sliding Window Counter Algorithm

Sliding window counter 是 fixed window counter 和 sliding window log 的混合方案。

核心思想：

- 使用当前窗口和上一个窗口的计数。
- 根据当前窗口经过的比例，对上一个窗口的计数加权。
- 估算当前滑动窗口内的请求数量。

公式：

```text
estimated_count =
  previous_window_count * overlap_ratio
  + current_window_count
```

截图中的例子：

- 限制：每分钟最多 7 个请求。
- 前一分钟有 5 个请求。
- 当前分钟到达 30% 位置。
- 当前分钟有 3 个请求。

计算：

```text
5 * 70% + 3 = 6.5
```

根据实现可以向下取整为 6，因此请求通过。

## 20. 图 4-11：Sliding Window Counter 示例

```mermaid
flowchart LR
    Prev["Previous window<br/>5 requests"]
    Curr["Current window<br/>3 requests<br/>30% elapsed"]
    Calc["Estimated count<br/>5 * 70% + 3 = 6.5"]
    Decision{"<= 7?"}
    Pass["Allow"]
    Reject["Reject"]

    Prev --> Calc
    Curr --> Calc
    Calc --> Decision
    Decision -- "Yes" --> Pass
    Decision -- "No" --> Reject
```

## 21. Sliding Window Counter 优缺点

优点：

- 平滑 fixed window 的突刺。
- 内存效率好。
- 容易理解。
- 比 sliding window log 更省内存。

缺点：

- 只是近似值，不是绝对精确。
- 对流量分布有假设，某些场景会有误差。

## 22. 算法对比

| 算法 | 优点 | 缺点 | 适合场景 |
|---|---|---|---|
| Token bucket | 简单、省内存、允许 burst | 需要调 bucket size 和 refill rate | 通用 API 限流 |
| Leaky bucket | 输出平滑、固定处理速率 | burst 时排队或丢弃，可能增加延迟 | 需要稳定处理速率 |
| Fixed window | 简单、省内存 | 窗口边界会突刺 | 粗粒度限流 |
| Sliding window log | 准确 | 耗内存 | 精确限流、低流量 |
| Sliding window counter | 平滑、省内存 | 近似估算 | 大多数需要折中的场景 |

## 23. 面试复习点

- Token bucket 最常用，允许 burst。
- Leaky bucket 输出速率固定。
- Fixed window 简单，但窗口边界有突刺。
- Sliding window log 准确但耗内存。
- Sliding window counter 在准确性和内存之间折中。
- 算法选择要结合业务能否接受 burst、是否要求精确、系统流量规模和内存限制。
