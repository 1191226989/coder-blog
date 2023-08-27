---
title: 微服务-Request-ID
created: '2023-07-24T14:15:50.986Z'
modified: '2023-08-27T11:03:24.421Z'
---

# 微服务-Request-ID

1. 客户端访问 web 服务时如何将该请求与服务器端日志关联
2. 微服务架构系统的访问日志如何查询
3. 不同项目交互异常，如何做日志关联

没有 Request-ID 的请求，只能根据调用函数记录日志关键字定位，在根据用户的输入的参数和时间来确定相关的日志

如果项目是以分布式微服务架构来实现的，代码多次封装后无法通过记录日志关键字与用户请求关联；微服务架构的用户请求逻辑层会拆分为多个子任务给下层服务处理，下层服务无法与用户请求关联；不同项目交互在并发和错误重试参数相同的情况下，也很难通过日志关键字和时间来定位问题

一般来说，在一个完整的请求中（对外暴露的是一个接口，对内的话可能经过 N 多个子服务），每个子服务共用一个相同的、全局唯一的 Request ID，这样当出现问题时根据 Request ID 就可以检索到请求当时的各个子服务的日志

应用 Request ID 需要有一套健全的日志系统

Request-ID 也可以配合 mysql 的唯一索引实现接口调用的幂等

#### Golang Web 日志记录 Request-ID
采用中间件形式，以 gin 框架为例

1. RequestId 中间件，主要的作用是生成 requestId
```go
func RequestIdMiddleware() gin.HandlerFunc {
    return func(ctx *gin.Context){
        requestId := ctx.Request.Header.Get("X-Request-Id")
        if requestId == "" {
            requestId = uuid.NewV4().String()
            ctx.Request.Header.Set("X-Request-Id", requestId)
        }

        // 写入响应
        ctx.Header("X-Request-Id", requestId)
        ctx.Next()
    }
}
```

2. 日志中间件
```go
func LogMiddleware() gin.HandlerFunc {
 return func(ctx *gin.Context){
     requestId := ctx.Request.Header.Get("X-Request-Id")
     ctx.Next()

     logrus.WithField("X-Request-Id", requestId).Info("xxxx")
 }
}
```
3. 使用方式
```go
engine.Use(RequestIdMiddleware(), LogMiddleware())
```

以 go-gin-api 为例 `https://github.com/xinliangnote/go-gin-api`
trace_id 链路 ID，String，例如：4b4f81f015a4f2a01b00 在中间件中进行设置，示例代码：
```go
// 如果存在 traceId 就使用原来的，如果不存在就重新生成。
if traceId := context.GetHeader(trace.Header); traceId != "" {
	context.setTrace(trace.New(traceId))
} else {
	context.setTrace(trace.New(""))
}
```

go-zero 链路追踪 `https://go-zero.dev/cn/docs/deployment/trace/`

微服务架构中调用链可能很漫长，从 http 到 rpc ，又从 rpc 到 http 。而开发者想了解每个环节的调用情况及效率，最佳方案就是`全链路跟踪`。

追踪的方法就是在一个请求开始时生成一个自己的 spanID ，随着整个请求链路传下去。我们则通过这个 spanID 查看整个链路的情况和性能问题。

#### HTTP
```go
func TracingHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // **1**
        carrier, err := trace.Extract(trace.HttpFormat, r.Header)
        // ErrInvalidCarrier means no trace id was set in http header
        if err != nil && err != trace.ErrInvalidCarrier {
            logx.Error(err)
        }

        // **2**
        ctx, span := trace.StartServerSpan(r.Context(), carrier, sysx.Hostname(), r.RequestURI)
        defer span.Finish()
        // **5**
        r = r.WithContext(ctx)

        next.ServeHTTP(w, r)
    })
}

func StartServerSpan(ctx context.Context, carrier Carrier, serviceName, operationName string) (
    context.Context, tracespec.Trace) {
    span := newServerSpan(carrier, serviceName, operationName)
    // **4**
    return context.WithValue(ctx, tracespec.TracingKey, span), span
}

func newServerSpan(carrier Carrier, serviceName, operationName string) tracespec.Trace {
    // **3**
    traceId := stringx.TakeWithPriority(func() string {
        if carrier != nil {
            return carrier.Get(traceIdKey)
        }
        return ""
    }, func() string {
        return stringx.RandId()
    })
    spanId := stringx.TakeWithPriority(func() string {
        if carrier != nil {
            return carrier.Get(spanIdKey)
        }
        return ""
    }, func() string {
        return initSpanId
    })

    return &Span{
        ctx: spanContext{
            traceId: traceId,
            spanId:  spanId,
        },
        serviceName:   serviceName,
        operationName: operationName,
        startTime:     timex.Time(),
        // 标记为server
        flag:          serverFlag,
    }
}

```

1. 将 header -> carrier，获取 header 中的 traceId 等信息
2. 开启一个新的 span，并把「traceId，spanId」封装在 context 中
3. 从上述的 carrier「也就是 header」获取 traceId，spanId
    - 看 header 中是否设置
    - 如果没有设置，则随机生成返回
4. 从 request 中产生新的 ctx，并将相应的信息封装在 ctx 中返回
5. 从上述的 context，拷贝一份到当前的 request

#### RPC
在 rpc 中存在 client/server ，所以从 tracing 上也有 clientTracing, serverTracing
```go
func TracingInterceptor(ctx context.Context, method string, req, reply interface{},
    cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    // open clientSpan
    ctx, span := trace.StartClientSpan(ctx, cc.Target(), method)
    defer span.Finish()

    var pairs []string
    span.Visit(func(key, val string) bool {
        pairs = append(pairs, key, val)
        return true
    })
    // **3** 将 pair 中的data以map的形式加入 ctx
    ctx = metadata.AppendToOutgoingContext(ctx, pairs...)

    return invoker(ctx, method, req, reply, cc, opts...)
}

func StartClientSpan(ctx context.Context, serviceName, operationName string) (context.Context, tracespec.Trace) {
    // **1**
    if span, ok := ctx.Value(tracespec.TracingKey).(*Span); ok {
        // **2**
        return span.Fork(ctx, serviceName, operationName)
    }

    return ctx, emptyNoopSpan
}
```
1. 获取上游带下来的 span 上下文信息
2. 从获取的 span 中创建新的 ctx，span「继承父 span 的 traceId」
3. 将生成 span 的 data 加入 ctx，传递到下一个中间件，流至下游

go-zero 通过拦截请求获取链路 traceID，然后在中间件函数入口会分配一个根 Span，然后在后续操作中会分裂出子 Span，每个 span 都有自己的具体的标识，Finsh 之后就会汇集在链路追踪系统中。开发者可以通过 ELK 工具追踪 traceID ，看到整个调用链。

同时 go-zero 并没有提供整套 trace 链路方案，开发者可以封装 go-zero 已有的 span 结构，做自己的上报系统，接入 jaeger, zipkin 等链路追踪工具。


