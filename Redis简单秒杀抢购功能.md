---
title: Redis简单秒杀抢购功能
created: '2023-08-26T07:38:03.395Z'
modified: '2023-08-26T07:42:31.751Z'
---

# Redis简单秒杀抢购功能

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8"
)

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:        "localhost:6379",
		Password:    "",
		DB:          0,
		IdleTimeout: 350,
		PoolSize:    50, // 连接池连接数量
	})
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()

	_, err := rdb.Ping(ctx).Result() // 检查连接redis是否成功
	if err != nil {
		fmt.Println("Redis Connect Failed: %v \n", err)
		panic(err)
	}

	ctx = context.Background()
	err = rdb.SetNX(ctx, "nums", 2, 0).Err()
	if err != nil {
		panic(err)
	}
	fmt.Println("set nums : ", 2)

	// Incr
	val, err := rdb.Incr(ctx, "nums").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("incr: ", val)

	// IncrBy
	val, err = rdb.IncrBy(ctx, "nums", 10).Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("incrby: ", val)

	//Decr
	val, _ = rdb.Decr(ctx, "nums").Result()
	fmt.Println("desc: ", val)

	//DecrBy
	val, _ = rdb.DecrBy(ctx, "nums", 5).Result()
	fmt.Println("decrby: ", val)
}
```

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8"
)

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:        "localhost:6379",
		Password:    "",
		DB:          0,
		IdleTimeout: 350,
		PoolSize:    50, // 连接池连接数量
	})
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()

	_, err := rdb.Ping(ctx).Result() // 检查连接redis是否成功
	if err != nil {
		fmt.Println("Redis Connect Failed: %v \n", err)
		panic(err)
	}

	ctx = context.Background()

	// Incr
	val, err := rdb.Incr(ctx, "nums").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("incr: ", val)

  if (val >= 10) {
    // 已经被抢光
  } else {
    //抢到了
    //保存队列或者直接保存数据库
  }
```
