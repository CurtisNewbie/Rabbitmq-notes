# MQ Notes

src:
- [关于 RabbitMQ Bottlenecks](https://blog.rabbitmq.com/posts/2014/04/finding-bottlenecks-with-rabbitmq-3-3)
- [Queuing Theory: Throughput, Latency and Bandwidth](https://blog.rabbitmq.com/posts/2012/05/some-queuing-theory-throughput-latency-and-bandwidth)

## 为什么用 MQ

1. 异步提升性能
    - 主要避免长时间阻塞, 等待结果的操作, 异步确保资源 (服务, CPU 等) 能充分利用, 在结果返回时对结果进行响应和下一部操作
    - 部分操作可在收到请求的时候直接返回, 随后在对请求进行处理, 如果后续请求处理失败, 我们更新对应结果状态即可 

2. 削峰/限流
    - `Producer` 生成的请求大于 `Consumer` 可处理的量, 导致 `Consumer` 打满甚至挂掉
    - 可以将受到请求转移到 MQ 中, 通过控制 MQ consume 的速率, 避免业务系统压力过大, 这种做法把压力从业务系统转移到了 MQ 上
    - 例如, RabbitMQ 使用 `prefetch (QOS)` 控制单个 `consumer` 最大可传输且未 `ack` 的消息的数量, 这里说的 `consumer`是指单个 `@RabbitListener`. 当 QOS 限制到达, 且消息未 `ack` 时, broker 会停止发送更多消息到 `consumer` 中, 可以认为 QOS 配置的是每个 `consumer` 对应的 `BlockingQueue`. 同时结合 listener container 的 `concurrency` 配置来控制 consumer 的数量，从而达到削峰的目的. 
    - 注意当 QOS 提高到一定程度的时候, 并不能继续提高性能, 因为 bottlenecks 更多在于网络的传输

3. 降低系统耦合性
    - 消息发送方和接收方通过 MQ 进行互动, 不存在接口传参等耦合的代码. 同时因为是异步操作, 更不需要关注调用时机等, 即使接收方暂时不可用也不会影响消息发送方的正常运作, 实现更灵活.


## 可能带来的问题

1. 消息顺序问题
    - 如果要严格遵守消息发送和接受的顺序, 我们需要保证单一 `queue`, 单一 `consumer`, 和单一 `channel` (Spring 中,每个 `consumer` 都会有自己的 `channel`, 所以单一 `consumer` 也保证单一 `channel`, `channel` 是不能在不同的线程之间共享的).
    - 同时，如果我们要确保每个消息按顺序完成处理, 而不止是消息接受的顺序, 我们打开 `ack`, 然后设置 `prefetch = 1`, 那就确保只有当前消息处理完毕才拿下一条消息.

2. 消息重复消费问题
    - 消费者需要记录消息, 实现应为 idempotent 的 

3. 可用性问题
    - MQ 作为额外需要运维的组件

4. 数据一致性问题
    - MQ 消费方可能没有正确的更新数据, 导致消息发送方与消费方数据不一致
