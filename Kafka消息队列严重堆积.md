---
title: Kafka消息队列严重堆积
created: '2023-08-26T06:30:59.045Z'
modified: '2023-08-26T06:55:35.750Z'
---

# Kafka消息队列严重堆积

#### 消息堆积
例如，积压了 100 万条，有 3 个 Consumer，每一个每秒能处理 200 条，3 个 Consumer 每秒一共能处理 600 条。 大概需要一个多小时才能处理完。

这一个多小时又会积压新的消息

所以正常处理肯定不行，需要提速。

例如 Kafka，这个消息积压的 Topic 有 3 个 Partition，那最多就能用 3 个 Consumer，所以增加 Consumer 没有用。

还是可以使用**临时队列**的方式。

```mermaid
graph LR

    subgraph 以前的 Topic
    Partition1(Partition 1)
    Partition2(Partition 2)
    Partition3(Partition 3)
    end

    subgraph 以前的 消费者
    Consumer1(Consumer 1)
    Consumer2(Consumer 2)
    Consumer3(Consumer 3)
    end

    Partition1-->Consumer1
    Partition2-->Consumer2
    Partition3-->Consumer3
    

    subgraph 临时的 Topic
    Partition01(Partition 1)
    Partition02(Partition 2)
    Partition03(Partition 3)
    Partition04(Partition 4)
    Partition05(Partition 5)
    Partition06(Partition 6)
    Partition07(Partition 7)
    Partition08(Partition 8)
    Partition09(Partition 9)
    end

    subgraph 临时的 消费者
    Consumer01(Consumer 1)
    Consumer02(Consumer 2)
    Consumer03(Consumer 3)
    Consumer04(Consumer 4)
    Consumer05(Consumer 5)
    Consumer06(Consumer 6)
    Consumer07(Consumer 7)
    Consumer08(Consumer 8)
    Consumer09(Consumer 9)
    end

    Partition01-->Consumer01
    Partition02-->Consumer02
    Partition03-->Consumer03
    Partition04-->Consumer04
    Partition05-->Consumer05
    Partition06-->Consumer06
    Partition07-->Consumer07
    Partition08-->Consumer08
    Partition09-->Consumer09


    Consumer1-->Partition01
    Consumer2-->Partition02
    Consumer3-->Partition03
    Consumer1-->Partition04
    Consumer2-->Partition05
    Consumer3-->Partition06
    Consumer1-->Partition07
    Consumer2-->Partition08
    Consumer3-->Partition09
```
新建一个 临时 Topic，设置为 10 个 Partition。

以前旧的 3 个 Consumer 不再处理业务逻辑了，改为只负责搬运，把消息分发放到临时 Topic 中（或者停掉旧的 3 个 Consumer，再新建 3 个 Consumer 只做分发逻辑）。

临时 Topic 这 10 个 Partition 可以有 10 个 新建 Consumer 了，它们来处理原来的业务逻辑。

这 10 个 Consumer 每秒一共能处理 2000 条了，这样十几分钟就可以处理完积压的 100 万条。

之后，再把整体结构恢复为原来的形式。

#### 消息被丢弃
首先，要实现防止消息过期问题，消息不应该设置过期时间。

如果就是设置了过期时间导致了消息丢失，怎么补救呢？

那就只能在访问量最低的时候，写一个临时程序来补消息了。

例如有订单消息丢了，那就需要找出哪些订单消息丢了，然后重新发到队列。

