# 08. 消息队列：面试题与答案

## Meta：设计 LeetCode

**参考答案：**

1. 提交接口持久化 submission_id、代码引用、语言和 pending 状态，再可靠发布判题任务。客户端按提交 ID 查询结果。
2. worker 在沙箱中编译运行，限制 CPU、内存、时间和网络；结果落库后确认消息。重复任务按 submission_id 幂等处理。
3. API 无状态扩容，worker 根据队列等待时间扩容；不同语言或资源需求可分队列。
4. 比赛排名依据可信判题结果和规则更新读模型；监控积压、判题失败与比赛峰值。

[题目来源](https://www.reddit.com/r/leetcode/comments/1lvnd0t/)

## OpenAI：设计 Webhook 回调系统

**参考答案：**

1. 保存订阅地址和策略，为每次投递生成稳定 delivery_id，写入持久队列，由 worker 发送签名 HTTP 请求。
2. 超时与暂时性失败按指数退避加抖动重试，超过期限进入死信队列；接收方按事件 ID 幂等处理重复投递。
3. 按目标限制并发与速率，隔离慢接收方；要求顺序时按订阅或业务实体协调消费。
4. 记录投递状态并支持受控重放；校验目标地址以防 SSRF，使用签名和时间戳校验来源。

[题目来源](https://www.reddit.com/r/leetcode/comments/1lvnd0t/)
