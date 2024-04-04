#### 单机版

```go
package utils

import "time"

var LimitQueue map[string][]int64

func LimitFreqSingle(queueName string, count uint, timeWindow int64) bool {
    currTime := time.Now().Unix()
    if LimitQueue == nil {
        LimitQueue = make(map[string][]int64)
    }
    if _, ok := LimitQueue[queueName]; !ok {
        LimitQueue[queueName] = make([]int64, 0)
    }
    // 队列未满
    if uint(len(LimitQueue[queueName])) < count {
        LimitQueue[queueName] = append(LimitQueue[queueName], currTime)
        return true
    }

    // 队列已满，取出最早访问的时间
    earlyTime := LimitQueue[queueName][0]
    // 最早的时间还在时间窗口范围内，不允许通过
    if currTime-earlyTime <= timeWindow {
        return false
    }else{
        // 最早访问时间已经超出了时间窗口
        LimitQueue[queueName] = LimitQueue[queueName][1:0]
        LimitQueue[queueName] = append(LimitQueue[queueName], currTime)
        return true
    }
}
```
```go
func limitIpFreq(ctx *gin.Context, timeWindow int64, count uint) bool {
    ip := ctx.ClientIP()
    key := "limit:" + ip 
    if !utils.LimitFreqSingle(key, count, timeWindow) {
        c.JSON(200, gin.H{
            "code": 400,
            "msg": "error: current ip frequetly visited",
        })
        return false
    }
    return true
}
```

#### redis版

```go
func LimitFreqs(queueName string, count uint, timeWindow int64) bool {
    currTime := time.Now().Unix()
    length := uint(utils.ListLen(queueName))
    if length < count {
        utils.ListPush(queueName, currTime)
        return true
    }
    // 队列满了，取出最早访问的时间
    earlyTime, _ := strconv.ParseInt(utils.ListIndex(queueName, int64(length)-1, 10, 64))
    
    if currTime - earlyTime <= timeWindow {
        return false
    }else{
        utils.ListPop(queueName)
        utils.ListPush(queueName, currTime)
    }
    return true
}
```