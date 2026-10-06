# 08. 消息队列：美国大厂面试题补充

[对应原笔记](08-message-queue.md) · [返回章节目录](README.md)

## 本节覆盖

照片异步处理、生产消费解耦、独立扩展、持久化、重试、幂等、顺序、死信和积压。

## 来源与题目标签

- **公开反馈真题**：Hello Interview 作者根据收到的候选人反馈整理的 2025 年题目，属于面试反馈汇总，非公司官方题库。原帖未逐题确认面试发生在美国，也未公开可计算频率的样本量。
- **本节练习追问**：依据原笔记整理的题干、英文口述版与回答要点；关联某道真题不代表该追问已被公司问过。
- **高频备考优先级**：强调可在多道设计题中复用的考点，不代表经统计验证的出题排名；不要当作 2026 年最新题库。

## 相关公开反馈真题

### Meta：设计 LeetCode / Design LeetCode

**标签：公开反馈真题（2025）。** [来源：候选人反馈汇总](https://www.reddit.com/r/leetcode/comments/1lvnd0t/)

**与本节的对应关系（学习映射）：** 异步判题队列、提交状态和 worker 扩展；还需代码沙箱与资源隔离。

### OpenAI：设计 Webhook 回调系统 / Design a Webhook Callback System

**标签：公开反馈真题（2025）。** [来源：候选人反馈汇总](https://www.reddit.com/r/leetcode/comments/1lvnd0t/)

**与本节的对应关系（学习映射）：** 回调异步投递、重试和幂等；还需签名验证、接收方容量与投递状态。

## 优先练习的面试追问

以下均为本节练习题，英文用于口述训练，不是公开面经逐字引文。

### Q1. 上传照片后要压缩、审核、生成缩略图，如何设计？

**英文口述：** Design an asynchronous photo-processing pipeline.

**继续追问：** 什么时候返回成功？队列里放图片还是引用？

**回答要点：** 持久化图片和任务，队列通常传对象引用与任务 ID，worker 执行并更新状态。区分上传成功和处理完成，提供查询或完成通知。

### Q2. worker 处理成功，在确认消息前崩溃，会怎样？

**英文口述：** What if a worker crashes after processing but before acknowledgement?

**继续追问：** 重复投递如何避免重复业务效果？

**回答要点：** 消息可能重新投递；使用稳定任务 ID 与持久化幂等记录，协调业务写入与完成记录。不能只靠内存集合去重，也不能把队列保证等同于外部副作用恰好一次。

### Q3. 生产速度长期高于消费速度，如何处理？

**英文口述：** How would you handle a growing queue backlog?

**继续追问：** 只增加 worker 是否有效？

**回答要点：** 队列仅缓冲峰值。看最老消息等待时间、生产/消费速率和下游容量；按瓶颈扩容、限流、背压或降级，避免扩 worker 压垮 DB。

### Q4. 失败消息如何重试？

**英文口述：** How would you design retries and a dead-letter queue?

**继续追问：** 接收方恢复前，重试会不会放大故障？

**回答要点：** 区分可重试和永久失败，指数退避加抖动、限制次数、投递死信并告警；重放仍必须幂等。

### Q5. 同一订单事件有顺序要求，如何并行处理？

**英文口述：** How would you preserve per-entity event ordering?

**继续追问：** 增加消费者会不会破坏顺序？

**回答要点：** 按订单 ID 路由到同一有序分区，设计分区内处理顺序和版本校验。全局顺序代价更高，热点订单仍可能限制并行度。

### Q6. 数据库写成功但发消息失败，怎么办？

**英文口述：** How would you avoid a database-and-queue dual-write failure?

**继续追问：** 重试发消息会不会重复？

**回答要点：** 补充点：在同一数据库事务写业务记录和 outbox，由 relay 可靠发布；relay 仍可能重复发布，所以消费者要幂等。

## 技术参考

[AWS 官方：SQS 至少一次投递与幂等](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)。这是技术机制参考，不是面试频率或公司出题证明。

## 口述自测

- 能否在两分钟内讲清问题、方案和代价？
- 能否说明正常请求路径和一个失败时序？
- 能否说明原笔记已覆盖哪些内容、完整设计还需补哪些知识？
