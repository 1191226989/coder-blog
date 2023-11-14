---
title: Redis分布式锁-go-redsyn
created: '2023-10-04T08:05:20.492Z'
modified: '2023-11-14T09:19:30.440Z'
---

# Redis分布式锁-go-redsyn

`go-redsyn` 实现了一个 `Mutex` 锁而且用到了 `redis` 连接池，代码也很简洁明了 `https://github.com/go-redsync/redsync`

```go
// A Mutex is a distributed mutual exclusion lock.
type Mutex struct {
	name   string
	expiry time.Duration

	tries     int
	delayFunc DelayFunc

	factor float64

	quorum int

	genValueFunc func() (string, error)
	value        string
	until        time.Time

	pools []redis.Pool
}
```

测试配置一个互斥锁：
```go
// NewMutex returns a new distributed mutex with given name.
func (r *Redsync) NewMutex(name string, options ...Option) *Mutex {
	m := &Mutex{
		name:   name,
		expiry: 8 * time.Second,
		tries:  32,
		delayFunc: func(tries int) time.Duration {
			return time.Duration(rand.Intn(maxRetryDelayMilliSec-minRetryDelayMilliSec)+minRetryDelayMilliSec) * time.Millisecond
		},
		genValueFunc: genValue,
		factor:       0.01,
		quorum:       len(r.pools)/2 + 1,
		pools:        r.pools,
	}
	for _, o := range options {
		o.Apply(m)
	}
	return m
}
```
```go
package main

import (
	"time"

	"github.com/go-redsync/redsync/v4"
	"github.com/go-redsync/redsync/v4/redis/redigo"
	redigolib "github.com/gomodule/redigo/redis"
	"github.com/stvp/tempredis"
)

func main() {
	server, err := tempredis.Start(tempredis.Config{})
	if err != nil {
		panic(err)
	}
	defer server.Term()

	pool := redigo.NewPool(&redigolib.Pool{
		MaxIdle:     3,
		IdleTimeout: 240 * time.Second,
		Dial: func() (redigolib.Conn, error) {
			return redigolib.Dial("unix", server.Socket())
		},
		TestOnBorrow: func(c redigolib.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	})

	rs := redsync.New(pool)

	//一般会根据实际业务增加配置锁的过期时间（默认为8秒）
	mutex := rs.NewMutex("test-redsync")

	if err = mutex.Lock(); err != nil {
		panic(err)
	}

	if _, err = mutex.Unlock(); err != nil {
		panic(err)
	}
}
```

redis 提供了 `INFO` 这个命令，能够随时监控服务器的状态 
```sh
127.0.0.1> info
```

在输出的信息里面有这几项说明缓存的状态：
```sh
keyspace_hits:14414110  
keyspace_misses:3228654  
used_memory:433264648  
expired_keys:1333536  
evicted_keys:1547380
```
通过计算 `hits` 和 `miss` 可以得到缓存的命中率：`14414110 / (14414110 +3228654) = 81%` ，一个缓存失效机制和过期时间设计良好的系统命中率可以做到 `95%` 以上。

尽可能的聚焦在**高频访问**且**时效性要求不高**的热点业务上（如字典数据、session、token），通过缓存预加载（预热）、合理调整缓存有效期的时间 （避免同时失效）、增加存储容量、调整缓存粒度、更新缓存等手段来提高命中率。

对于时效性很高（或缓存空间有限），内容跨度很大（或访问很随机），并且访问量不高的应用来说缓存命中率可能长期很低。

