# Go 语言学习路线规划（面向 Java/Spring Boot 开发者）

> 适用读者：具备 Java + Spring Boot 分布式系统开发经验，希望系统掌握 Go 语言的后端工程师。
>
> 核心思路：**用已知映射未知**，以 Java 概念为锚点，快速建立 Go 的思维模型。
>
> 预计周期：**9-12 周**，达到能独立开发生产级 Go 微服务的水平。

---

## 🗺️ 全局路线图

```
Phase 1: 语法迁移          (Week 1-2,  约 14 天)
       ↓
Phase 2: 核心特性 - 并发    (Week 3-4,  约 14 天)
       ↓
Phase 3: 工程实践           (Week 5-6,  约 14 天)
       ↓
Phase 4: 微服务与分布式     (Week 7-9,  约 21 天)
       ↓
Phase 5: 深入底层 & 高级特性 (Week 10+, 持续学习)
```

---

## 🔑 Java → Go 核心概念速查表

| Java 概念 | Go 对应 | 关键差异 |
|---|---|---|
| `class` | `struct` | Go 无继承，只有组合 |
| `interface` | `interface` | Go 是**隐式实现**，无需 `implements` |
| `try/catch/finally` | `err` 多返回值 + `defer` | Go 用多返回值处理错误，`defer` 替代 finally |
| `ArrayList<T>` | `[]T`（slice） | 动态数组，内置原语 |
| `HashMap<K,V>` | `map[K]V` | 原生支持，非线程安全 |
| `Optional<T>` | `*T` + `nil` 判断 | 指针即可选值 |
| `ThreadLocal` | `context.Context` | Go 不推荐 thread-local，用 context 传递 |
| Spring `@Bean` / IoC | 手动注入 / `wire` | Go 无框架级 IoC 容器 |
| `synchronized` | `sync.Mutex` | 显式互斥锁 |
| `ReentrantReadWriteLock` | `sync.RWMutex` | 读写锁 |
| `CountDownLatch` | `sync.WaitGroup` | 等待一组 goroutine 完成 |
| `CompletableFuture` | `goroutine + channel` | 原生并发原语，更轻量 |
| `ExecutorService` | Worker Pool（goroutine + channel） | 手动实现或用第三方库 |
| `@Transactional` | 手动事务管理 | Go 无注解，需显式控制 |
| `pom.xml` | `go.mod` | Go Module 依赖管理 |
| `mvn package` | `go build` | 编译为原生二进制，无 JVM |

---

## Phase 1：语法迁移期（Week 1-2）

> **目标**：能独立写出结构清晰的 Go 程序，消除语法陌生感，建立 Go 的错误处理哲学。

### Day 1-2：基础语法速通

**学习内容：**
1. 环境安装与配置（Go 安装、GOPROXY 设置、go mod init）
2. Hello World 解剖（package、import、func main）
3. 变量声明的四种方式（`var`、类型推断、`:=`、批量声明）
4. 基础数据类型（int/float/bool/string/byte）
5. 类型转换（Go 是强类型，必须显式转换）
6. 控制流：`if`（含初始化语句）、`for`（Go 唯一循环）、`switch`（无需 break）
7. 函数：多返回值、命名返回值、函数作为一等公民、闭包
8. 错误处理哲学：`error` 接口、`fmt.Errorf` 包装、`errors.Is` 解包

**重点对比：**
- `var x = 5` vs `x := 5`（短变量声明只能在函数体内使用）
- Go 变量声明后必须使用，否则编译报错
- Go 的 `for range` 遍历 map 是**无序**的
- `%w` 在 `fmt.Errorf` 中的作用：包装错误，支持 `errors.Is` 解包

**课后练习：**
- 实现 `parseNumbers(strs []string) ([]int, error)` 函数
- 实现 `statistics(nums []int) (min, max int, avg float64, err error)` 函数
- 串联两个函数，处理所有错误情况

---

### Day 3-4：struct + interface + 方法

**学习内容：**
1. struct 定义与初始化（字面量、`new`、指针接收者 vs 值接收者）
2. 方法绑定（`func (s *MyStruct) Method()`）
3. 嵌入（Embedding）—— Go 的"组合替代继承"
4. interface 隐式实现原理（无需 `implements` 关键字）
5. 空接口 `interface{}`（等价于 Java `Object`）与 `any`（Go 1.18+）
6. 类型断言 `x.(T)` 和类型判断 `switch x.(type)`
7. 接口组合（多个 interface 组合成新 interface）

**重点对比：**

```
Java 实现接口：                    Go 实现接口：
class Dog implements Animal {      type Dog struct{}
    public void Speak() {...}      func (d Dog) Speak() string {...}
}                                  // 自动满足 Animal 接口，无需声明
```

**课后练习：**
- 定义 `Shape` interface（含 `Area() float64` 方法）
- 实现 `Circle`、`Rectangle` 两个 struct
- 编写 `PrintArea(s Shape)` 函数，体验多态

---

### Day 5-7：切片、Map 与指针

**学习内容：**
1. 数组（Array）vs 切片（Slice）——Go 中几乎只用 Slice
2. Slice 的底层结构（ptr + len + cap）
3. `make`、`append`、`copy` 操作
4. Slice 的陷阱：共享底层数组，修改会相互影响
5. Map 的创建、遍历、删除、并发不安全性
6. 指针基础（`&` 取地址、`*` 解引用）
7. 何时用指针接收者，何时用值接收者

**重点对比：**

| Java | Go |
|---|---|
| `new ArrayList<>()` | `make([]int, 0, 10)` |
| `list.add(x)` | `slice = append(slice, x)` |
| `new HashMap<>()` | `make(map[string]int)` |
| `map.remove(key)` | `delete(m, key)` |
| `map.containsKey(k)` | `v, ok := m[k]; if ok {...}` |

**课后练习：**
- 实现一个简单的 LRU Cache（用 map + 双向链表，或直接用 map 模拟）
- 体会 slice append 扩容时的内存行为

---

### Day 8-9：包管理与项目结构

**学习内容：**
1. `go.mod` 详解（module 路径、依赖版本、replace 指令）
2. `go get`、`go tidy`、`go vendor` 命令
3. 包（package）的可见性规则（大写=导出，小写=包私有）
4. 推荐项目目录结构（`cmd/`、`internal/`、`pkg/`）
5. `init()` 函数的执行时机

**标准项目结构：**

```
myapp/
  cmd/
    server/
      main.go          ← 程序入口
  internal/            ← 包私有（类比 Java package-private）
    service/
    repository/
    model/
    handler/
  pkg/                 ← 可被外部引用的公共包
    errors/
    utils/
  configs/
    config.yaml
  go.mod
  go.sum
  Makefile
```

---

### Day 10-12：defer / panic / recover

**学习内容：**
1. `defer` 的执行时机（函数返回前，LIFO 顺序）
2. `defer` 与闭包的交互（参数在 defer 时求值，还是执行时？）
3. `panic` 的使用场景（真正不可恢复的错误）
4. `recover` 恢复 panic（必须在 defer 中调用）
5. `panic/recover` vs Java `RuntimeException/catch` 的哲学差异

**重点规则：**
- `defer` 多个时是 **LIFO（后进先出）**
- `defer` 的参数在**声明时**就被求值（非执行时）
- 不要用 `panic` 做正常的错误处理，只用于"程序不应该运行到这里"的情况

---

### Day 13-14：综合实战

**实战项目：用 Go 重写一个熟悉的 Spring Boot CRUD REST API**

要求：
- 不使用任何框架（纯标准库 `net/http`）
- 实现用户的增删改查（内存存储，无需数据库）
- 严格的错误处理（每个错误都有恰当的 HTTP 状态码）
- 合理的项目分层（handler / service / repository）

目的：
- 对比两者代码量和结构差异
- 体会 Go 标准库的能力边界
- 为后续引入 Gin 框架做铺垫

---

## Phase 2：核心特性 - 并发编程（Week 3-4）

> **目标**：掌握 Go 最核心的竞争优势，建立并发思维，能正确使用 goroutine + channel。

### Day 15-16：Goroutine 基础

**学习内容：**
1. Goroutine 启动（`go func()`）
2. Goroutine vs OS Thread vs Java Thread 的本质区别
3. Go 调度器（GMP 模型）概念理解（不需要深入，了解即可）
4. Goroutine 的内存占用（约 2-8KB 栈，可动态增长）
5. Goroutine 泄漏的常见原因和排查

**核心对比：**

| | Java Thread | Goroutine |
|---|---|---|
| 创建成本 | ~1MB 栈内存 | ~2KB 栈内存 |
| 数量上限 | 数千（受OS限制） | 百万级（Go Runtime 调度） |
| 通信方式 | 共享内存 + 锁 | Channel（推荐）或共享内存 |
| 创建语法 | `new Thread(r).start()` | `go func(){}()` |

---

### Day 17-18：Channel

**学习内容：**
1. Channel 的创建（无缓冲 vs 有缓冲）
2. 发送（`ch <- v`）和接收（`v := <-ch`）
3. `close(ch)` 和 `for range ch` 遍历
4. `select` 语句（多路复用，类比 Java `Selector`）
5. 单向 Channel（`chan<-` 和 `<-chan`）
6. Channel 的常见模式：Pipeline、Fan-Out、Fan-In

**经典 Pipeline 模式：**

```go
// 生产者 → 处理器 → 消费者
// 完全对应 Spring Batch 的 Reader → Processor → Writer
producer(data) → processChannel → consumer(results)
```

---

### Day 19-20：sync 包与并发安全

**学习内容：**
1. `sync.Mutex` / `sync.RWMutex`
2. `sync.WaitGroup`（等待 goroutine 完成）
3. `sync.Once`（单例初始化，类比 Spring `@Lazy` 单例）
4. `sync.Map`（并发安全 Map）
5. `sync/atomic` 包（原子操作）
6. 竞态检测：`go run -race`（必学工具！）

---

### Day 21-22：Context

**学习内容：**
1. `context.Background()` 和 `context.TODO()`
2. `context.WithTimeout`、`context.WithDeadline`、`context.WithCancel`
3. Context 的传播链（父 Context 取消，子 Context 自动取消）
4. Context 传值（`WithValue`）的正确使用场景
5. 在 HTTP 服务、数据库操作、RPC 调用中正确传递 Context

**核心原则：**
- Context 必须作为函数的**第一个参数**，命名为 `ctx`
- 不要把 Context 存在 struct 中
- Context 传值只用于请求级别的元数据（如 traceID、userID）

---

### Day 23-24：并发模式实战

**实战项目：并发爬虫/数据处理器**

要求：
- 实现 Worker Pool 模式（N 个 goroutine 处理 M 个任务）
- 使用 Context 实现超时控制
- 使用 WaitGroup 等待所有任务完成
- 使用 channel 收集所有结果
- 用 `-race` 检测无竞态条件

---

### Day 25-28：并发深入

**学习内容：**
1. Channel 的内存模型（happens-before 关系）
2. 常见并发 Bug 排查（死锁、goroutine 泄漏）
3. `errgroup`（`golang.org/x/sync/errgroup`）—— WaitGroup 的升级版
4. `semaphore`（信号量，控制并发数）

---

## Phase 3：工程实践（Week 5-6）

> **目标**：能构建生产级的 Go HTTP 服务，掌握主流生态工具链。

### 技术选型对照表

| Spring Boot 生态 | Go 生态（推荐） | 说明 |
|---|---|---|
| Spring MVC | **Gin** | 最流行的 HTTP 路由框架 |
| Spring Data JPA | **GORM** | 全功能 ORM |
| Spring Data JPA | **sqlx** | 轻量级 SQL 映射（性能更好） |
| Logback/SLF4J | **Zap**（uber-go） | 高性能结构化日志 |
| `application.yml` | **Viper** | 配置管理（支持 yaml/env） |
| Spring Cache + Redis | **go-redis** | Redis 客户端 |
| Spring DI / Wire | **Wire**（google） | 依赖注入代码生成 |
| Spring Validation | **go-validator** | 参数校验 |
| Spring Test | `testing` + **testify** | 单元测试 |
| Spring Boot Actuator | **prometheus/client_golang** | 监控指标暴露 |

### Day 29-32：Gin + 中间件

**学习内容：**
1. Gin 路由、路由组、路径参数
2. 请求绑定（`ShouldBindJSON`、`ShouldBindQuery`）
3. 中间件机制（`c.Next()` 类比 `chain.doFilter()`）
4. 统一错误处理中间件
5. 日志中间件（接入 Zap）
6. JWT 认证中间件
7. 跨域（CORS）中间件

---

### Day 33-36：数据库 + 配置 + 日志

**学习内容：**
1. GORM 基本操作（CRUD、关联、事务）
2. GORM Hooks（类比 JPA `@PrePersist`）
3. Viper 加载 `config.yaml`，支持环境变量覆盖
4. Zap 结构化日志（JSON 格式，含 traceID 字段）
5. 依赖注入：用 Wire 组装 handler → service → repository

---

### Day 37-40：测试与 Docker 部署

**学习内容：**
1. Go 单元测试（`testing` 包，`_test.go` 文件）
2. Table-Driven Test（Go 最佳实践）
3. Mock（使用 `testify/mock` 或 `gomock`）
4. 集成测试（使用 `testcontainers-go` 启动真实 DB）
5. 编写 `Dockerfile`（多阶段构建，最终镜像约 10MB）
6. `Makefile` 自动化构建流程

**多阶段 Dockerfile 模板：**

```dockerfile
# 构建阶段
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

# 运行阶段（最终镜像极小）
FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
```

---

## Phase 4：微服务与分布式（Week 7-9）

> **目标**：将 Spring Cloud 的分布式经验迁移到 Go 生态，构建多服务协作系统。

### Spring Cloud → Go 微服务生态对照

| Spring Cloud | Go 生态 | 说明 |
|---|---|---|
| OpenFeign | **gRPC** / go-resty | 服务间通信 |
| Eureka / Nacos | **Consul** / etcd | 服务注册与发现 |
| Spring Cloud Config | **Consul KV** / Nacos | 动态配置中心 |
| Ribbon | gRPC 内置负载均衡 | 客户端负载均衡 |
| Hystrix / Sentinel | **go-resilience** / uber-go/ratelimit | 熔断限流 |
| Sleuth + Zipkin | **OpenTelemetry** + Jaeger | 分布式链路追踪 |
| Spring Cloud Gateway | **Kong** / APISIX / 自研 Gin 网关 | API 网关 |
| Spring Kafka | **sarama** / confluent-kafka-go | Kafka 客户端 |

### Day 41-45：gRPC

**学习内容：**
1. Protobuf 语法（`.proto` 文件定义）
2. 生成 Go 代码（`protoc` + `protoc-gen-go`）
3. 实现 gRPC Server 和 Client
4. gRPC 中间件（UnaryInterceptor）
5. gRPC + Context 超时传播
6. gRPC vs REST 的性能对比场景

---

### Day 46-49：服务注册与发现（Consul）

**学习内容：**
1. Consul 本地启动（Docker）
2. 服务注册（健康检查配置）
3. 服务发现（通过 Consul 找到目标服务地址）
4. 动态配置（Consul KV Watch）
5. 与 gRPC 集成，实现真正的微服务调用

---

### Day 50-54：可观测性三件套

**学习内容：**
1. **Metrics（指标）**：Prometheus + Grafana
   - 暴露 `/metrics` 端点
   - 自定义业务 Counter、Histogram
2. **Tracing（追踪）**：OpenTelemetry + Jaeger
   - 在 HTTP 和 gRPC 调用链中自动传播 TraceID
3. **Logging（日志）**：Zap + ELK/Loki
   - 结构化日志字段：traceID、spanID、userID、latency

---

### Day 55-63：综合实战 - 双服务微服务系统

**实战项目：订单服务 + 库存服务**

要求：
- `order-service`（HTTP + Gin）接收创建订单请求
- `inventory-service`（gRPC）处理库存扣减
- 两个服务通过 Consul 发现彼此
- 全链路 OpenTelemetry 追踪（在 Jaeger 可视化）
- Kafka 发布订单创建事件
- Docker Compose 一键启动全套环境

---

## Phase 5：深入底层 & 高级特性（Week 10+）

> **目标**：理解 Go Runtime 底层原理，能做性能调优，掌握高级语言特性。

### GC 与内存模型（对比 JVM）

| JVM | Go Runtime | 差异 |
|---|---|---|
| JVM Heap | Go Heap | Go GC 停顿 < 1ms（JVM G1 可能 > 10ms） |
| 栈扩展（固定大小） | 分段栈/连续栈（动态增长） | Goroutine 栈从 2KB 动态增长 |
| JIT 编译 | AOT 编译（静态二进制） | Go 无需预热，启动即峰值性能 |
| `-Xmx` 设置堆上限 | `GOGC` 控制 GC 触发阈值 | GOGC=100 表示堆增长 100% 时触发 GC |
| JVisualVM / Arthas | `pprof` | Go 内置，无需安装额外工具 |

### Day 64-70：性能分析与调优

**学习内容：**
1. `pprof` CPU/内存分析实战
2. 逃逸分析（`go build -gcflags="-m"`）
3. 减少内存分配（对象池 `sync.Pool`）
4. 字符串拼接优化（`strings.Builder`）
5. Benchmark 编写（`func BenchmarkXxx(b *testing.B)`）
6. `go test -bench=. -benchmem` 分析内存分配次数

---

### Day 71-77：泛型（Go 1.18+）

**学习内容：**
1. 类型参数语法 `[T any]`
2. 类型约束（interface 作为约束）
3. 内置约束：`comparable`、`constraints.Integer` 等
4. 泛型函数、泛型 struct
5. 泛型的使用场景与限制（不要过度使用）

---

### Day 78+：持续深入方向

根据实际工作需要选择：

- **网络编程**：`net` 包、WebSocket、自定义 TCP 协议
- **CGO**：调用 C 库、性能敏感场景
- **编译器插件**：`go generate`、代码生成
- **云原生**：Kubernetes Operator 开发（client-go）
- **数据库驱动开发**：理解 `database/sql` 接口设计

---

## 📚 推荐资源

### 书籍（按阅读顺序）

| 书名 | 适用阶段 | 说明 |
|---|---|---|
| 《Go程序设计语言》 | Phase 1-2 | 圣经级入门，Alan Donovan 著 |
| 《Go语言高并发与微服务实战》 | Phase 2-3 | 结合分布式背景 |
| 《100 Go Mistakes and How to Avoid Them》 | Phase 3-4 | 有 Java 背景必读，避坑手册 |
| 《Go语言设计与实现》 | Phase 5 | 深入 Runtime 原理（骚金）|

### 在线资源

| 资源 | 链接 | 用途 |
|---|---|---|
| Go Tour | tour.golang.org | 官方交互教程，1天速通语法 |
| Go by Example | gobyexample.com | 代码片段速查 |
| Effective Go | go.dev/doc/effective_go | 官方最佳实践 |
| awesome-go | github.com/avelino/awesome-go | 最全 Go 库清单 |
| Go Playground | play.golang.org | 在线运行 Go 代码 |

### 工具链

| 工具 | 用途 | 类比 Java |
|---|---|---|
| `go fmt` | 代码格式化 | Checkstyle |
| `go vet` | 静态检查 | SpotBugs |
| `golangci-lint` | 全面 Lint | SonarQube |
| `go test -race` | 竞态检测 | ThreadSanitizer |
| `go tool pprof` | 性能分析 | JVisualVM / Arthas |
| `dlv` (Delve) | 调试器 | IntelliJ Debugger |

---

## ⚡ 给 Java 开发者的关键心智转变

1. **错误不是异常**：每一个 `error` 都要显式处理，不能 `catch` 后不管
2. **组合优于继承**：Go 没有继承，用 struct embedding 实现复用，用 interface 实现多态
3. **interface 是隐式的**：不需要声明"我实现了谁"，只要方法签名匹配就行
4. **并发是原生的**：goroutine 和 channel 不是库，是语言内置原语
5. **没有 null 安全**：Go 的 `nil` 无处不在，养成防御性指针判断习惯
6. **没有 JVM 预热**：Go 编译为静态二进制，启动即满速，适合 Serverless
7. **简单胜于优雅**：Go 刻意不支持泛型（直到 1.18）、不支持方法重载，追求可读性

---

## 🎯 里程碑检验标准

| 阶段 | 完成标志 |
|---|---|
| Phase 1 完成 | 能用纯标准库写出有完整错误处理的 HTTP CRUD API |
| Phase 2 完成 | 能正确实现 Worker Pool，用 `-race` 检测无竞态 |
| Phase 3 完成 | 能用 Gin + GORM + Zap 搭建生产级服务并容器化 |
| Phase 4 完成 | 能搭建 2 个微服务通过 gRPC + Consul 通信，带全链路追踪 |
| Phase 5 完成 | 能用 pprof 定位并优化内存/CPU 热点 |
