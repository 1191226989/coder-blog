---
title: Redis实现连续签到-bitmap
created: '2023-12-23T11:16:31.251Z'
modified: '2024-01-07T09:03:28.727Z'
---

# Redis实现连续签到-bitmap

[https://redis.io/docs/data-types/bitmaps](https://redis.io/docs/data-types/bitmaps)

位图`BitMap`也就是 `byte` 数组，用二进制表示只有 `0` 和 `1` 两个数字。

Redis提供的数据类型`BitMap`（位图），每个bit位对应`0`和`1`两个状态。内部还是采用`String`类型存储，提供了一些指令用于直接操作`BitMap`，可以把它看作一个bit数组，数组的下标就是偏移量。

它的优点是内存开销小，效率高且操作简单，很适合用于签到这类场景。缺点在于位计算和位表示数值的局限。

1. 基于最小的单位bit进行存储，所以非常省空间。
2. 设置时候时间复杂度O(1)、读取时候时间复杂度O(n)，操作是非常快的。
3. 二进制数据的存储，进行相关计算的时候非常快。
4. 方便扩容

redis中bit映射被限制在512MB之内，所以最大是2^32位。建议每个key的位数不能过大，因为读取时候时间复杂度O(n)，越大的串读的时间花销越多。

字符串`big`对应的二进制代码为`01100010 01101001 01100111`
`key: value[011000100110100101100111]`

#### 重要命令
|命令|定义|
|------|--------------|
|`getbit key offset`|对key所存储的字符串值，获取指定偏移量上的位（bit）|
|`setbit key offset value`|对key所存储的字符串值，设置或清除指定偏移量上的位（bit）1. 返回值为该位在setbit之前的值 2. value只能取0或1 3. offset从0开始，即使原位图只能10位，offset可以取1000|
|`bitcount key [start end [BYTE | BIT]]`|获取位图指定范围中位值为1的个数  如果不指定start与end，则取所有 By default, the additional arguments start and end specify a byte index. We can use an additional argument BIT to specify a bit index. |
|`bitop op destKey key1 [key2...]`|做多个BitMap的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存在destKey中|
|`bitpos key targetBit [start end]`|计算位图指定范围第一个偏移量对应的的值等于targetBit的位置  1. 找不到返回-1  2. start与end没有设置，则取全部  3. targetBit只能取0或者1 |

```sh
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> getbit hello 0
(integer)0
127.0.0.1:6379> getbit hello 1
(integer)1
127.0.0.1:6379> setbit hello 7 1
(integer)0
127.0.0.1:6379> get hello
"cig"
127.0.0.1:6379> bitcount hello
(integer)13
127.0.0.1:6379> bitcount hello 1 3
(integer)9
127.0.0.1:6379> set world small
OK
127.0.0.1:6379> bitop and hello:and:world hello world
(integer)5
127.0.0.1:6379> bitcount hello:and:world
(integer)11
127.0.0.1:6379> bitpos hello 1
(integer)1
127.0.0.1:6379> bitpos hello 0 1 2
(integer)8
```

- 空间占用、以及第一次分配空间需要的时间 <来自Redis文档>
在一台2010MacBook Pro上，`offset`为`2^32 -1`（分配512MB）需要～`300ms`，`offset`为`2^30 -1`(分配128MB)需要～`80ms`，`offset`为`2^28 -1`（分配32MB）需要～`30ms`，`offset`为`2^26 -1`（分配8MB）需要`8ms`。大概的空间占用计算公式是：(`$offset/8/1024/1024`)MB

#### 使用场景一：用户签到
```go
import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "", // no password set
    DB:       0,  // use default DB
})

// 用户uid
userId := 110

// 记录有uid的key
cacheKey := fmt.Sprintf("sign_%d", userId)

// 开始有签到功能的日期
startDate := "2023-01-01"

// 今天的日期
todayDate := "2023-01-21"

// 计算offset
layout := "2006-01-02"
startTime, err := time.Parse(layout, startDate)
if err != nil {
  fmt.Println(err)
  return
}
todayTime, err := time.Parse(layout, todayDate)
if err != nil {
  fmt.Println(err)
  return
}
offset = math.Floor((todayTime - startTime) / 86400)

fmt.Printf("今天是第%d天 \n", offset)

// 签到
// 一年一个用户大约365/8=45.625个字节
err := rdb.SetBit(ctx, cacheKey, offset, 1).Err()
if err != nil {
    panic(err)
}

// 查询签到情况
val, err := rdb.GetBit(ctx, cacheKey, offset).Result()
if err != nil {
    panic(err)
}
fmt.Printf("签到状态是%d \n", val)

// 计算总签到次数
val, err := rdb.BitCount(ctx, cacheKey, nil).Result()
if err != nil {
    panic(err)
}

// 计算某段时间内的签到次数
// https://redis.io/commands/bitcount
// By default, the additional arguments start and end specify a byte index. We can use an additional argument BIT to specify a bit index.
val, err := rdb.BitCount(ctx, cacheKey, &redis.BitCount{
  Start:startTime,
  End:todayTime,
}).Result()
if err != nil {
    panic(err)
}

```

#### 使用场景二：统计活跃用户
```go
import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "", // no password set
    DB:       0,  // use default DB
})

//模拟日期对应的活跃用户
userList := map[string][]int{
  "2023-01-10": []int{1,2,3,4,5,6,7,8,9,10},
  "2023-01-11": []int{1,2,3,4,5,6,7,8},
  "2023-01-12": []int{1,2,3,4,5,6},
  "2023-01-13": []int{1,2,3,4},
  "2023-01-14": []int{1,2},
}
for date, userIds range userList {
  cacheKey := fmt.Sprintf("stat_%s", date)
  for _, uid range userids {
    err := rdb.SetBit(ctx, cacheKey, uid, 1).Err()
    if err != nil {
        fmt.Println(err)
        continue
    }
  }
}

val, err := rdb.BitOpAnd(ctx, "stat", "stat_2023-01-10", "stat_2023-01-11", "stat_2023-01-12").Result()
if err != nil {
   fmt.Println(err)
   return
}
fmt.Println(val)

val, err := rdb.BitCount(ctx, "stat", nil).Result()
if err != nil {
    panic(err)
}
fmt.Println(val)

val, err := rdb.BitOpAnd(ctx, "stat1", "stat_2023-01-13", "stat_2023-01-14", "stat_2023-01-15").Result()
if err != nil {
   fmt.Println(err)
   return
}
fmt.Println(val)

val, err := rdb.BitCount(ctx, "stat1", nil).Result()
if err != nil {
    panic(err)
}
fmt.Println(val)

```
假设当前站点有**5000W**用户，那么一天的数据大约为`50000000/8/1024/1024=6MB`

#### 使用场景三：用户签到
1. 签到1天得1积分，连续签到2天得2积分，3天得3积分，3天以上均得3积分等。
2. 如果连续签到中断，则重置计数，每月重置计数。
3. 显示用户某月的签到次数和首次签到时间。
4. 在日历控件上展示用户每月签到，可以切换年月显示。

考虑到每月要重置连续签到次数，最简单的方式是按用户每月存一条签到数据。Key的格式为 u:sign:{uid}:{yyyMM}，而Value则采用长度为4个字节的（32位）的BitMap（最大月份只有31天）。BitMap的每一位代表一天的签到，1表示已签，0表示未签。

例如 u:sign:1225:202301 表示ID=1225的用户在2023年1月的签到记录
```sh
# 用户1月6号签到
SETBIT u:sign:1225:202301 5 1 # 偏移量是从0开始，所以要把6减1

# 检查1月6号是否签到
GETBIT u:sign:1225:202301 5 # 偏移量是从0开始，所以要把6减1

# 统计1月份的签到次数
BITCOUNT u:sign:1225:202301

# 获取1月份前31天的签到数据
BITFIELD u:sign:1225:202301 get u31 0

# 获取1月份首次签到的日期
BITPOS u:sign:1225:202301 1 # 返回的首次签到的偏移量，加上1即为当月的某一天
```

#### 使用场景四：统计每天用户的登录数

每一位标识一个用户ID，当某个用户访问网页或执行了某个操作，就在`bitmap`中把标识此用户的位设置为1。

使用 `set` 和 `BitMap` 存储的对比

1. 1亿用户，5千万独立

|数据类型|每个userid占用空间|需要存储的用户量|全部内存量|
|-------|-----------------|---------------|---------|
|set|32位（假设userid用的是整型，实际很多网站用的是长整型）|50,000,000|32位 * 50,000,000 = 200 MB|
|bitmap|1位|100,000,000|1 位 * 100,000,000 = 12.5 MB|

|数据类型|一天|一个月|一年|
|-------|-----------------|---------------|---------|
|set|200M|6G|72G|
|bitmap|12.5M|375M|4.5G|

2. 只有 10 万独立用户

|数据类型|每个userid占用空间|需要存储的用户量|全部内存量|
|-------|-----------------|---------------|---------|
|set|32位（假设userid用的是整型，实际很多网站用的是长整型）|1,000,000|32位 * 1,000,000 = 4 MB|
|bitmap|1位|100,000,000|1 位 * 100,000,000 = 12.5 MB|
如果独立用户数量很多，使用 BitMap 明显更有优势，能节省大量的内存。但如果独立用户数量较少，还是建议使用 set 存储，BitMap 会产生多余的存储开销。

注意：`setbit` 时较大的偏移量，可能有较大耗时

#### 使用场景五：视频属性的无限延伸
需求分析：一个拥有亿级数据量的短视频app，视频存在各种属性(是否加锁、是否特效等等)，需要做各种标记。

可能想到的解决方案：

1. 如果存储在mysql中，随着业务增长属性可能一直增加，并且存在有时间限制的属性，直接对数据库进行加减字段容易锁库。即使是保存在一个字段中用json等压缩技术存储也存在读效率的问题，并且对于大于几亿的数据来说，废弃的字段回收起来非常困难。

2. 记录保存在redis中，根据业务属性＋uid为key来存储。读写效率高，但是存储的角度来说key的数据量大于value了，太耗费空间，并且几亿的数据回收也是难题。

设计方案：
- 使用redis的bitmap进行存储。
- key由属性id＋视频分片id组成。
- value按照视频id对分片范围取模来决定偏移量offset。
- 10亿视频一个属性约`120m`还是挺划算的。
- 后续增加新属性也只是business_id再增加一个值。

分片原因：1.读取的时候时间复杂度是`O(n)`存储越长读取时间越多 2.bitmap有长度限制`2^32`。

分片粒度：1.如果主键id存在断层，则尽可能选择的粒度可以避开此段id范围，防止空间浪费。2.分片粒度可参考某一单位时间的增长值来判断，这样也有利于预算占了多少空间，虽然空间不会占很多。

#### 使用场景六：用户在线状态
设计方案：
使用bitmap是一个节约空间效率又高的一种方法，只需要一个key，然后用户id为偏移量offset，如果在线就设置为`1`，不在线就设置为`0`，**3亿**用户只需要`36MB`的空间。

```go
import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "", // no password set
    DB:       0,  // use default DB
})

err := rdb.SetBit(ctx, "online", userId, 1).Err()
if err != nil {
    fmt.Println(err)
}

val, err := rdb.GetBit(ctx, "online", userId).Result()
if err != nil {
    panic(err)
}
fmt.Println(val)

```
可以增加分片。10亿数据过大。10w分一片。

#### 使用场景七：统计活跃用户
设计方案：
使用时间作为缓存的key，然后用户id为offset，如果当日活跃过就设置为1。之后通过bitOp进行二进制计算统计在某段时间内用户的活跃情况。

```go
import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "", // no password set
    DB:       0,  // use default DB
})

err := rdb.SetBit(ctx, "active_20230708", userId, 1).Err()
if err != nil {
    fmt.Println(err)
}
err := rdb.SetBit(ctx, "active_20230709", userId, 1).Err()
if err != nil {
    fmt.Println(err)
}

val, err := rdb.BitOpAnd(ctx, "active_total", "active_20230708", "active_20230709").Result()
if err != nil {
   fmt.Println(err)
   return
}
fmt.Println(val)

```
上亿用户需要增加分片。

