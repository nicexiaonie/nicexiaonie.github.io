# 一、Golang 核心技术（5年经验）

## 1. 并发编程

**1. 请详细解释Goroutine的调度机制（GMP模型）？**

**答案：**
GMP模型是Go运行时的核心调度模型：
- **G (Goroutine)**：代表一个goroutine，包含栈、指令指针、调度信息
- **M (Machine)**：代表一个操作系统线程，执行G
- **P (Processor)**：代表调度的上下文，维护G的本地队列，M必须绑定P才能执行G

**调度流程：**
1. 每个P维护一个本地G队列（最多256个）
2. M从绑定的P的本地队列获取G执行
3. 本地队列为空时，从全局队列或其他P偷取G（work stealing）
4. G执行系统调用时，M会解绑P，P可绑定其他M继续执行
5. G阻塞时会被放回队列，M继续执行其他G

**优势：**
- 减少锁竞争（本地队列无锁）
- work stealing实现负载均衡
- 抢占式调度防止G长时间占用

**2. Channel的底层实现原理是什么？有缓冲和无缓冲Channel的区别？**

**答案：**
**底层结构（hchan）：**
```go
type hchan struct {
    qcount   uint           // 当前队列中的元素个数
    dataqsiz uint           // 环形队列大小
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 元素大小
    closed   uint32         // 是否关闭
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 接收等待队列
    sendq    waitq          // 发送等待队列
    lock     mutex          // 互斥锁
}
```

**无缓冲Channel：**
- dataqsiz = 0，没有缓冲区
- 发送和接收必须同时准备好才能完成（同步）
- 发送方会阻塞直到接收方接收
- 用于同步通信

**有缓冲Channel：**
- dataqsiz > 0，有环形队列缓冲
- 缓冲未满时发送不阻塞，缓冲未空时接收不阻塞
- 用于异步通信和削峰

**操作流程：**
- 发送：检查recvq是否有等待的G，有则直接发送；否则放入buf或sendq
- 接收：检查sendq是否有等待的G，有则直接接收；否则从buf取或放入recvq

**3. 如何避免Goroutine泄漏？请举例说明常见的泄漏场景**

**答案：**
**常见泄漏场景：**

1. **Channel发送阻塞：**
```go
// 错误：接收方退出，发送方永久阻塞
func leak1() {
    ch := make(chan int)
    go func() {
        ch <- 1 // 永久阻塞
    }()
    // 主goroutine退出，但发送goroutine泄漏
}

// 正确：使用带缓冲channel或context
func fixed1(ctx context.Context) {
    ch := make(chan int, 1)
    go func() {
        select {
        case ch <- 1:
        case <-ctx.Done():
            return
        }
    }()
}
```

2. **HTTP请求未设置超时：**
```go
// 错误：请求永不超时
resp, _ := http.Get("http://slow-server.com")

// 正确：设置超时
client := &http.Client{Timeout: 5 * time.Second}
resp, _ := client.Get("http://slow-server.com")
```

3. **互斥锁未释放：**
```go
// 错误：panic导致锁未释放
func leak3() {
    var mu sync.Mutex
    mu.Lock()
    panic("error") // 锁未释放
    mu.Unlock()
}

// 正确：使用defer
func fixed3() {
    var mu sync.Mutex
    mu.Lock()
    defer mu.Unlock()
    // 业务逻辑
}
```

4. **WaitGroup使用错误：**
```go
// 错误：Add和Done不匹配
var wg sync.WaitGroup
wg.Add(1)
go func() {
    // 忘记调用Done()
}()
wg.Wait() // 永久阻塞
```

**避免方法：**
- 使用Context控制goroutine生命周期
- 设置超时时间
- 使用defer确保资源释放
- 使用pprof检测goroutine泄漏

**4. Context包的作用是什么？如何正确使用Context进行超时控制和取消操作？**

**答案：**
**Context的作用：**
1. 传递请求范围的数据（如traceID）
2. 控制goroutine的生命周期
3. 超时控制
4. 取消信号传播

**四种Context：**
```go
// 1. 根Context（不可取消）
ctx := context.Background()
ctx := context.TODO()

// 2. 可取消Context
ctx, cancel := context.WithCancel(parent)
defer cancel() // 确保资源释放

// 3. 超时Context
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// 4. 截止时间Context
ctx, cancel := context.WithDeadline(parent, time.Now().Add(5*time.Second))
defer cancel()
```

**正确使用示例：**
```go
func doWork(ctx context.Context) error {
    // 创建带超时的子context
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    // 启动工作goroutine
    resultCh := make(chan string, 1)
    go func() {
        // 模拟耗时操作
        time.Sleep(2 * time.Second)
        resultCh <- "done"
    }()

    // 等待结果或超时
    select {
    case result := <-resultCh:
        return nil
    case <-ctx.Done():
        return ctx.Err() // 返回超时或取消错误
    }
}

// HTTP服务中使用
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // 获取请求的context
    if err := doWork(ctx); err != nil {
        http.Error(w, err.Error(), http.StatusRequestTimeout)
    }
}
```

**最佳实践：**
- Context作为第一个参数传递
- 不要存储Context在结构体中
- 不要传递nil Context，使用context.TODO()
- Context.Value只用于请求范围的数据，不用于可选参数
- 总是调用cancel函数释放资源

**5. sync.Mutex和sync.RWMutex的区别？什么时候使用读写锁？**

**答案：**
**sync.Mutex（互斥锁）：**
- 同一时刻只允许一个goroutine访问
- 适用于读写都需要互斥的场景
- 简单高效

**sync.RWMutex（读写锁）：**
- 允许多个goroutine同时读
- 写操作独占，与读写互斥
- 适用于读多写少的场景

**性能对比：**
```go
// Mutex：所有操作都互斥
var mu sync.Mutex
mu.Lock()
// 读或写操作
mu.Unlock()

// RWMutex：读操作可并发
var rwmu sync.RWMutex

// 读操作（可并发）
rwmu.RLock()
// 读取数据
rwmu.RUnlock()

// 写操作（独占）
rwmu.Lock()
// 修改数据
rwmu.Unlock()
```

**使用场景：**
- **使用Mutex：**
  - 读写比例接近
  - 临界区很小
  - 简单场景

- **使用RWMutex：**
  - 读操作远多于写操作（如缓存）
  - 读操作耗时较长
  - 需要提高并发读性能

**示例：**
```go
type Cache struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // 读锁，允许并发读
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key, val string) {
    c.mu.Lock()          // 写锁，独占访问
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**注意事项：**
- RWMutex比Mutex开销大，读少写多时反而慢
- 不能在持有读锁时升级为写锁（会死锁）
- 写锁会等待所有读锁释放

**6-10题：sync.WaitGroup、select、Goroutine池、happens-before、atomic包**

见主文件详细答案（题6-10）

## 2. 内存管理与GC

**11-18题：内存分配、GC算法、逃逸分析、内存对齐、内存泄漏排查、sync.Pool、栈管理**

见主文件详细答案（题11-18）

## 3. 数据结构与类型系统

**19-26题：slice、map、interface、类型断言、defer、panic/recover、值传递**

见主文件详细答案（题19-26）

## 4. 网络编程

**27-33题：net/http、HTTP服务器、gRPC、WebSocket、TCP粘包、IO模型、连接池**

见主文件详细答案（题27-33）

## 5. 性能优化

**34-39题：pprof、性能瓶颈、CPU优化、内存优化、benchmark、内联优化**

见主文件详细答案（题34-39）
