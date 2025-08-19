## EFrame 中间件最小用例手册（精简版）

说明：示例均基于本仓库封装，无新增第三方库；聚焦“最小可用”，可直接复制调整运行；每段代码包含必要注释。

### 日志 Logger（zap 封装）

```go
package main

import (
    "context"
    "go.uber.org/zap"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    var logger *library.Log
    cfg := &library.LoggerConfig{ // 基础配置
        Receiver:       &logger,
        Level:          "info",
        Name:           "app",
        Path:           "./logs",
        MaxAgeDay:      7,
        MaxFileSize:    100,
        MaxBackups:     10,
        DebugMode:      true,
        PrintInConsole: true,
        ServiceName:    "demo",
        CompressFile:   true,
    }
    logger = library.GetLogger(cfg)

    logger.Info("启动成功") // 基础日志
    logger.Error("错误示例", zap.String("reason", "demo"))

    // 带上下文日志（自动带 trace_id 等）
    ctx := context.Background()
    logger.InfoCtx(ctx, "处理业务", zap.String("uid", "123"))
}
```

### Gin 中间件：请求日志 / 异常恢复 / SkyWalking

```go
package main

import (
    "github.com/gin-gonic/gin"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/plugin/gin_plugin"
)

func main() {
    var logger *library.Log
    logger = library.GetLogger(&library.LoggerConfig{
        Receiver:       &logger,
        Level:          "info",
        Name:           "http",
        Path:           "./logs",
        PrintInConsole: true,
        ServiceName:    "demo",
    })

    // SkyWalking 可选
    var sky *library.SkyWalking
    sky, _ = library.NewSkyWalkingTracker(&library.SkyWalkingConfig{Receiver: &sky, ServiceName: "demo", Addr: "127.0.0.1:11800", Sample: 1.0})
    sky.SwitchTrace(true)

    r := gin.New()
    r.Use(gin_plugin.PanicRecovery(logger, true))           // panic 恢复
    r.Use(gin_plugin.RequestLogMiddleware(logger, 1<<20))   // 请求日志，1MB 限制
    r.Use(gin_plugin.SkyWalkingReport(r, sky))              // SkyWalking 打点（可选）

    r.GET("/ping", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "ok"}) })
    _ = r.Run(":8080")
}
```

### Redis（go-redis v8 封装）

```go
package main

import (
    "context"
    "time"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    ctx := context.Background()
    rc, err := library.NewRedisClient(ctx, &library.RedisConfig{
        ConnectionName:   "default",
        Addr:             "127.0.0.1",
        Port:             6379,
        DB:               0,
        PoolSize:         20,
        EnableSkyWalking: true,
        DialTimeout:      5 * time.Second,
        ReadTimeout:      3 * time.Second,
        WriteTimeout:     3 * time.Second,
        MinIdleConns:     5,
        IdleTimeout:      5 * time.Minute,
    })
    if err != nil { panic(err) }

    _ = rc.Set(ctx, "k", "v", time.Hour).Err()
    val, _ := rc.Get(ctx, "k").Result()
    _, _ = rc.Del(ctx, "k").Result()
    _ = val
}
```

### GORM（MySQL）

```go
package main

import (
    "gorm.io/gorm"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

type User struct { ID uint `gorm:"primarykey"`; Name string `gorm:"size:64"` }

func main() {
    db, err := library.NewGormDB(&library.GormConfig{
        ConnectionName:    "default",
        DBName:            "test",
        Host:              "127.0.0.1",
        Port:              "3306",
        UserName:          "root",
        Password:          "pass",
        Charset:           "utf8mb4",
        ParseTime:         "True",
        Loc:               "Local",
        MaxLifeTime:       3600,
        MaxOpenConn:       50,
        MaxIdleConn:       10,
        EnableTransaction: true,
        EnableSkyWalking:  true,
        // ReadOnlySlavesConfigs: []library.GormSlaveConfig{{Host:"127.0.0.1", Port:"3307", UserName:"readonly", Password:"pass"}},
    })
    if err != nil { panic(err) }

    _ = db.AutoMigrate(&User{})
    _ = db.Transaction(func(tx *gorm.DB) error { return tx.Create(&User{Name: "Tom"}).Error })
}
```

### HTTP 客户端

```go
package main

import (
    "context"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    var logger *library.Log
    logger = library.GetLogger(&library.LoggerConfig{Receiver: &logger, Level: "info", Name: "httpc", Path: "./logs", PrintInConsole: true, ServiceName: "demo"})

    client := library.NewHttpClient(&library.HttpClientConfig{
        Name: "api", RequestTimeoutSecond: 15, DialTimeoutSecond: 5, DialKeepAliveSecond: 30,
        MaxIdleConnections: 100, MaxIdleConnectionsPerHost: 10, IdleConnTimeoutSecond: 90, EnableSkyWalking: true,
    })

    ctx := context.Background()
    _, _ = client.Get(ctx, "https://httpbin.org/get", map[string]string{"X-Demo": "1"}, logger)
    _, _ = client.Post(ctx, "https://httpbin.org/post", map[string]string{"Content-Type": "application/json"}, []byte(`{"a":1}`), logger)
}
```

### Kafka 生产者

```go
package main

import (
    "github.com/Shopify/sarama"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    p, err := library.NewKafkaSyncProducer(&library.KafkaProducerConfig{
        Name:          "p1",
        BrokerAddress: []string{"127.0.0.1:9092"},
        Version:       "2.6.0",
        ConsoleDebug:  true,
    })
    if err != nil { panic(err) }
    defer p.Close()

    _, _, _ = p.SendMessage(&sarama.ProducerMessage{Topic: "test-topic", Value: sarama.StringEncoder("hello")})
}
```

### Kafka 消费者（消费者组）

```go
package main

import (
    "context"
    "github.com/Shopify/sarama"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    c, err := library.NewKafkaGroupConsumer(&library.KafkaGroupConsumerConfig{
        Name: "c1", GroupName: "g1", Version: "2.6.0",
        Topics: []string{"test-topic"}, BrokerAddress: []string{"127.0.0.1:9092"}, InitialOffset: sarama.OffsetNewest, ConsoleDebug: true,
    })
    if err != nil { panic(err) }
    defer c.Close()

    c.SetMessageHandleFunc(func(msg *sarama.ConsumerMessage) error { return nil })      // 自动确认
    c.SetMessageHandleByHandFunc(func(msg *sarama.ConsumerMessage) error { return nil }) // 手动确认

    ctx := context.Background()
    for { _ = c.Consume(ctx) }
}
```

### 本地缓存（LRU + 过期 + 可选清理协程）

```go
package main

import (
    "time"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    cache := library.NewLocalCache(1024)                        // 基础缓存
    tickerCache := library.NewTickerLocalCache(1024, time.Minute) // 自动清理
    defer tickerCache.Close()

    _ = cache.Put("k", "v", 10*time.Second)
    if v, ok := cache.Get("k"); ok { _ = v }
    cache.Delete("k")
}
```

### Elasticsearch（v7）

```go
package main

import (
    "context"
    "github.com/olivere/elastic/v7"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    es, err := library.NewElasticV7(&library.ElasticV7Config{
        ConnectionName:      "es7",
        Addr:                "http://127.0.0.1:9200",
        Username:            "elastic",
        Password:            "pass",
        HealthCheckInterval: 10,
        IsGzip:              true,
    })
    if err != nil { panic(err) }
    defer es.Close()

    ctx := context.Background()
    _, _ = es.Index().Index("users").Id("1").BodyJson(map[string]any{"name": "Tom"}).Do(ctx)
    _, _ = es.Search().Index("users").Query(elastic.NewMatchAllQuery()).Do(ctx)
}
```

### MongoDB

```go
package main

import (
    "context"
    "go.mongodb.org/mongo-driver/bson"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    mc, err := library.NewMongoClient(&library.MongoConf{ConnectionName: "m1", Host: "127.0.0.1", Port: "27017", Timeout: 10})
    if err != nil { panic(err) }
    defer mc.Close()

    col := mc.GetDatabase("test").Collection("users")
    _, _ = col.InsertOne(context.Background(), bson.M{"name": "Tom"})
}
```

### HTTP Server（net/http 封装）

```go
package main

import (
    "net/http"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    srv := library.NewHttpServer(&library.HttpServerConfig{Port: 8080}, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        _, _ = w.Write([]byte("ok"))
    }))
    _ = srv.Run()  // 异步启动
    // ... 退出时 srv.Close()
}
```

### 环境变量 / 配置（Viper）

```go
package main

import (
    "fmt"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    env, err := library.NewEnv("config", "yaml", "./configs/")
    if err != nil { panic(err) }
    host := env.GetStringWithDefault("database.host", "127.0.0.1")
    port := env.GetIntWithDefault("database.port", 3306)
    fmt.Println(host, port)
}
```

### 命令行参数解析（CLI）

```go
package main

import (
    "context"
    "fmt"
    "talkcheap.xiaoeknow.com/xiaoetong/eframe/library"
)

func main() {
    cli := library.NewCliCommand("MyApp", "应用描述", "用法")
    cli.AddConfig(&library.CommandConfig{
        Signature:   "send-email {--to= : 收件人} {--subject= : 主题} {--body= : 内容}",
        Description: "发送邮件",
        HandleFunc: func(ctx context.Context, cmd *library.ExecCommand) error {
            fmt.Println(cmd.Options()["to"], cmd.Options()["subject"], cmd.Options()["body"])
            return nil
        },
    })
    if err := cli.Setup(); err != nil { panic(err) }
    if err := cli.Exec(context.Background()); err != nil { panic(err) }
}
```

### 定时任务调度（gocron）

```go
package main

import (
    "fmt"
    "time"
    "github.com/go-co-op/gocron"
)

func main() {
    s := gocron.NewScheduler(time.UTC)
    s.Every(5).Seconds().Do(func(){ fmt.Println("tick", time.Now().Format("15:04:05")) })
    s.StartAsync()
    select{} // 持续运行
}
```

---

小贴士：
- 如需统一依赖注入或生命周期管理，可结合 `Receiver` 字段与容器使用（本仓库已有封装）。
- 数据源仓储封装（`datasource`）提供 Query/分页/搜索的通用能力，详见 `datasource/README.md`。


