---
title: Golang延时任务队列go-queue
created: '2023-08-29T12:34:26.088Z'
modified: '2023-09-02T12:17:13.327Z'
---

# Golang延时任务队列go-queue

https://github.com/beanstalkd/beanstalkd

https://github.com/zeromicro/go-queue

#### 使用场景

1. 创建订单后多长时间没有付款，要取消订单，并退库存
2. 用户注册成功后，发送一封邮件
3. 定期检查退款状态的订单是否退款成功
4. 定期检查已付款订单第三方对账

#### 高可用
- 支持多实例部署。挂掉一个实例后，还有后备实例继续提供服务。
- go-queue对外提供的 API 使用 cluster 模型，内部将多个 node 封装起来，多个 node 之间冗余存储。

#### 其他方案
考虑过类似基于 kafka/rocketmq 等消息队列作为存储的方案，最后由于存储设计模型放弃了这类选择。

举个例子，假设以 Kafka 这种消息队列存储来实现延时功能，每个队列的时间都需要创建一个单独的 topic(如: Q1-1s, Q1-2s…)。这种设计在延时时间比较固定的场景下问题不太大，但如果是延时时间变化比较大会导致 topic 数目过多，会把磁盘从顺序读写会变成随机读写从导致性能衰减，同时也会带来其他类似重启或者恢复时间过长的问题。

1. topic 过多 → 存储压力
2. topic 存储的是现实时间，在调度时对不同时间 (topic) 的读取，顺序读 → 随机读
3. 同理，写入的时候顺序写 → 随机写

#### 架构设计
```mermaid
graph TB
    producer1("dq.NewProducer()")
    producer2("producer.Delay()")

    subgraph Node
    node1(node1)
    node2(node2)
    node3(node3)
    end

    subgraph Beanstalkd
    conn1(conn1)
    conn2(conn2)
    conn3(conn3)
    tube[tube]
    end

    producer1-->node1
    producer1-->node2
    producer1-->node3
    
    producer2 --> conn1
    producer2 --> conn2
    producer2 --> conn3

    node1.->conn1
    node2.->conn2
    node3.->conn3

    conn1-."Put(data)".->tube
    conn2-."Put(data)".->tube
    conn3-."Put(data)".->tube
```

#### API设计
- 生产者
1. `producer.At(msg []byte, at time.Time)`
2. `producer.Delay(body []byte, delay time.Duration)`
3. `producer.Revoke(ids string)`

- 消费者
1. `consumer.Consume(consume handler)`

使用延时队列后，队列中 job 的状态变迁：

1. service → `producer.At(msg []byte, at time.Time)` → 延时job写入到 tube 
2. 定时触发 → job 状态更新为 `ready`
3. consumer 获取到 `ready` job → 取出 job，开始消费；并更改状态为 `reserved`
4. 执行传入 consumer 中的 handler 逻辑处理函数`

#### 生产实践
- **生产端**
1. 开发中生产延时任务，**只需确定任务执行时间**

传入 At() `producer.At(msg []byte, at time.Time)`，内部会自行计算时间差值，写入 tube

2. 如果出现任务时间的修改，以及任务内容的修改

在生产时可能需要额外建立一个 logic_id → job_id 的关系表，查询到 job_id → `producer.Revoke(ids string)` ，对其删除，然后重新写入

- **消费端 (分为 cluster 和 node 模式)**

- cluster模式
> https://github.com/zeromicro/go-queue/blob/master/dq/consumer.go#L45

1. cluster 内部将 consume handler 做了一层再封装
2. 对 consume body 做hash，并使用此 hash 作为 redis 去重的key
3. 如果存在，则不做处理，丢弃

- node模式
> https://github.com/zeromicro/go-queue/blob/master/dq/consumernode.go#L36

1. 消费 node 获取到 ready job；先执行 Reserve(TTR)，预订此job，将执行该job进行逻辑处理
2. 在 node 中 delete(job)；然后再进行消费。如果失败，则上抛给业务层，做相应的兜底重试

对于消费端，开发者需要自己实现消费的幂等性。

#### consumer example
```go
consumer := dq.NewConsumer(dq.DqConf{
	Beanstalks: []dq.Beanstalk{
		{
			Endpoint: "localhost:11300",
			Tube:     "tube",
		},
		{
			Endpoint: "localhost:11300",
			Tube:     "tube",
		},
	},
	Redis: redis.RedisConf{ // 使用 redis 的 setNX 达到 job id 的唯一消费
		Host: "localhost:6379",
		Type: redis.NodeType,
	},
})
consumer.Consume(func(body []byte) {
	fmt.Println(string(body))
})
```
#### producer example

```go
producer := dq.NewProducer([]dq.Beanstalk{
	{
		Endpoint: "localhost:11300",
		Tube:     "tube",
	},
	{
		Endpoint: "localhost:11300",
		Tube:     "tube",
	},
})	

for i := 1000; i < 1005; i++ {
	_, err := producer.Delay([]byte(strconv.Itoa(i)), time.Second*5)
	if err != nil {
		fmt.Println(err)
	}
}
```
