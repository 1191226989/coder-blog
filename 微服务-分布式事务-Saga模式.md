---
title: 微服务-分布式事务-Saga模式
created: '2023-08-27T02:55:51.469Z'
modified: '2023-08-27T11:03:00.177Z'
---

# 微服务-分布式事务-Saga模式

### 1 Saga 相关概念
1987 年普林斯顿大学发表了一篇 Paper Sagas，讲述的是如何处理 long lived transaction（长活事务）。Saga 是一个长活事务可被分解成可以交错运行的子事务集合。其中每个子事务都是一个保持数据库一致性的真实事务。

#### 1.1 Saga 的组成
- 每个 Saga 由一系列 sub-transaction Ti 组成
- 每个 Ti 都有对应的补偿动作 Ci，补偿动作用于撤销 Ti 造成的结果

和 TCC 相比，Saga 没有 “预留” 动作，它的 Ti 就是直接提交到库

#### 1.2 Saga 的执行顺序有两种：

- T1, T2, T3, ..., Tn
- T1, T2, ..., Tj, Cj,..., C2, C1，其中 0 < j < n

#### 1.3 Saga 定义了两种恢复策略：
- backward recovery，向后恢复，**补偿所有已完成的事务**，如果任一子事务失败。即上面提到的第二种执行顺序，其中 j 是发生错误的 sub-transaction，这种做法的效果是撤销掉之前所有成功的 sub-transation，使得整个 Saga 的执行结果撤销。

- forward recovery，向前恢复，**重试失败的事务**，假设每个子事务最终都会成功。**适用于必须要成功的场景**，执行顺序是类似于这样的：T1, T2, ..., Tj (失败), Tj (重试),..., Tn，其中 j 是发生错误的 sub-transaction。该情况下不需要 Ci。
```mermaid
graph LR
    subgraph Normal-Transactions
    T1(T1)
    T2(T2)
    T3(T3)
    Tn(Tn)
    end
    
    subgraph Compensating-Transactions
    C3-.->C2
    C2-.->C1
    end

    T1==>T2
    T2==>T3
    T3==>Tn

    T3-.Failure.->C3
 ```   

向前恢复没有必要提供补偿事务，如果你的业务中子事务（最终）总会成功，或补偿事务难以定义或不可能，向前恢复更符合你的需求。

根据这两种实现，SAGA 可以分为两部分：

- 子事务（Normal Transactions）：大事务拆分若干个小事务，将整个事务 T 分解为 n 个子事务，命名为 T1、T2、…、Tn。每个子事务都应该是或者能被视为是原子行为。如果分布式事务能够正常提交，其对数据的影响（最终一致性）应与连续按顺序成功提交 Ti 等价。

- 补偿事务（Compensating Transactions）：每个子事务对应的补偿动作，命名为 C1、C2、…、Cn。

子事务与补偿动作需要满足一些条件：

1. Ti 与 Ci 必须对应
2. 补偿动作 Ci 一定会执行成功，即需要实现最大努力交付。
3. Ti 与 Ci 需要具备幂等性

理论上补偿事务永不失败，然而，在分布式世界中服务器可能会宕机，网络可能会失败，甚至数据中心也可能会停电。在这种情况下最后的手段是提供回退措施，比如**人工干预**。

#### 1.4 Saga 的使用条件

- Saga 只允许两个层次的嵌套，顶级的 Saga 和简单子事务
- 在外层，全原子性不能得到满足。也就是说，sagas 可能会看到其他 sagas 的部分结果
- 每个子事务应该是独立的原子行为
- 在我们的业务场景下，各个业务环境（如：航班预订、租车、酒店预订和付款）是自然独立的行为，而且每个事务都可以用对应服务的数据库保证原子操作。

补偿也有需考虑的事项： 补偿事务从语义角度撤消了事务 Ti 的行为，但未必能将数据库返回到执行 Ti 时的状态。（**例如，如果事务触发电邮发送， 则可能无法撤消此操作**） 但这对我们的业务来说不是问题。其实难以撤消的行为也有可能被补偿。例如，发送电邮的事务可以通过发送解释问题的另一封电邮来补偿。

#### 1.5 对于 ACID 的保证

Saga 对于 ACID 的保证和 TCC 一样：

- 原子性（Atomicity），正常情况下保证。
- 一致性（Consistency），在某个时间点，会出现 A 库和 B 库的数据违反一致性要求的情况，但是最终是一致的。
- 隔离性（Isolation），在某个时间点，A 事务能够读到 B 事务部分提交的结果。
- 持久性（Durability），和本地事务一样，只要 commit 则数据被持久。
- Saga 不提供 ACID 保证，因为原子性和隔离性不能得到满足。原论文描述如下：

> full atomicity is not provided. That is, sagas may view the partial results of other sagas

**通过 saga log，saga 可以保证一致性和持久性。**

#### 1.6 和 TCC 对比

Saga 相比 TCC 的缺点是缺少预留动作，导致补偿动作的实现比较麻烦：Ti 就是 commit，比如一个业务是发送邮件，在 TCC 模式下，先保存草稿（Try）再发送（Confirm），撤销的话直接删除草稿（Cancel）就行了。而 Saga 则就直接发送邮件了（Ti），如果要撤销则得再发送一份邮件说明撤销（Ci），实现起来有一些麻烦。

如果把上面的发邮件的例子换成：A 服务在完成 Ti 后立即发送 Event 到 ESB（企业服务总线，可以认为是一个消息中间件），下游服务监听到这个 Event 做自己的一些工作然后再发送 Event 到 ESB，如果 A 服务执行补偿动作 Ci，那么整个补偿动作的层级就很深。

不过没有预留动作也可以认为是优点：

有些业务很简单，套用 TCC 需要修改原来的业务逻辑，而 Saga 只需要添加一个补偿动作就行了。 TCC 最少通信次数为 2n，而 Saga 为 n（n=sub-transaction 的数量）。 有些第三方服务没有 Try 接口，TCC 模式实现起来就比较 tricky 了，而 Saga 则很简单。 没有预留动作就意味着不必担心资源释放的问题，异常处理起来也更简单（请对比 Saga 的恢复策略和 TCC 的异常处理）。

### 2 Saga 相关实现 Saga Log

Saga 保证所有的子事务都得以完成或补偿，但 Saga 系统本身也可能会崩溃。Saga 崩溃时可能处于以下几个状态：

- Saga 收到事务请求，但尚未开始。因子事务对应的微服务状态未被 Saga 修改，则什么也不需要做。
- 一些子事务已经完成。重启后，Saga 必须接着上次完成的事务恢复。
- 子事务已开始，但尚未完成。由于远程服务可能已完成事务，也可能事务失败，甚至服务请求超时，saga 只能重新发起之前未确认完成的子事务。这意味着子事务必须幂等。
- 子事务失败，其补偿事务尚未开始。Saga 必须在重启后执行对应补偿事务。
- 补偿事务已开始但尚未完成。解决方案与上一个相同。这意味着补偿事务也必须是幂等的。
- 所有子事务或补偿事务均已完成，与第一种情况相同。

为了恢复到上述状态，必须追踪子事务及补偿事务的每一步。通过事件的方式达到以上要求，并将以下事件保存在名为 saga log 的持久存储中：

- **Saga started event** 保存整个 saga 请求，其中包括多个事务 / 补偿请求 
- **Transaction started event** 保存对应事务请求 
- **Transaction ended event** 保存对应事务请求及其回复 
- **Transaction aborted event** 保存对应事务请求和失败的原因 
- **Transaction compensated event** 保存对应补偿请求及其回复 
- **Saga ended event** 标志着 saga 事务请求的结束，不需要保存任何内容

```mermaid
sequenceDiagram
  caller ->> saga: request
  caller -->> saga: saga started
  caller -->> saga: transaction started
  
  saga ->>+ database:T
  database ->>- saga:T

  caller -->> saga: transaction ended/aborted

  saga ->>+ database:C
  database ->>- saga:C

  caller -->> saga: transaction compensated
  
  saga ->> caller: response

  caller -->> saga: saga ended

 ```   

通过将这些事件持久化在 saga log 中，可以将 saga 恢复到上述任何状态。

由于 Saga 只需要做事件的持久化，而事件内容以 JSON 的形式存储，Saga log 的实现非常灵活，数据库（SQL 或 NoSQL），持久消息队列，甚至普通文件可以用作事件存储。

### 3 Golang分布式事务管理框架 
dtm `https://github.com/dtm-labs/dtm`
go-zero 微服务框架也整合了这个框架 `https://github.com/zeromicro/go-zero`

