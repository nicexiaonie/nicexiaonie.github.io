[toc]
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

**6. 深入理解 GMP 调度器的抢占机制**

**答案：**
Go 调度器经历了三个抢占阶段的演进：

**1. 协作式抢占（Go 1.2-1.13）：**
- 依赖函数调用时的栈增长检查
- 问题：无函数调用的死循环无法被抢占
```go
// 这段代码在 Go 1.13 前会阻塞其他 goroutine
func deadloop() {
    for {
        // 无函数调用，无法被抢占
    }
}
```

**2. 基于信号的异步抢占（Go 1.14+）：**
- 使用 SIGURG 信号实现真正的抢占
- sysmon 线程检测运行超过 10ms 的 G
- 向 M 发送抢占信号，强制切换

**抢占流程：**
```
1. sysmon 监控线程每 20μs-10ms 运行一次
2. 检测到 G 运行超过 10ms
3. 标记 G 的抢占标志 (g.preempt = true)
4. 向 M 发送 SIGURG 信号
5. M 收到信号，保存 G 的上下文
6. 将 G 放回队列，调度其他 G
```

**3. 基于协作的抢占点优化（Go 1.17+）：**
- 在循环中插入抢占检查点
- 编译器优化，减少性能损耗

**关键数据结构：**
```go
type g struct {
    preempt       bool    // 抢占标志
    preemptStop   bool    // 抢占到安全点
    preemptShrink bool    // 栈收缩抢占
}

type p struct {
    schedtick   uint32    // 每次调度递增
    syscalltick uint32    // 每次系统调用递增
}
```

**7. Work Stealing 算法的实现细节**

**答案：**
Work Stealing 是 GMP 模型实现负载均衡的核心机制。

**窃取策略：**
```go
// 调度器查找可运行 G 的顺序
func findrunnable() *g {
    // 1. 每调度 61 次，从全局队列获取（防止全局队列饥饿）
    if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
        return globrunqget(_g_.m.p.ptr(), 1)
    }

    // 2. 从本地队列获取
    if gp := runqget(_g_.m.p.ptr()); gp != nil {
        return gp
    }

    // 3. 从全局队列获取
    if sched.runqsize > 0 {
        return globrunqget(_g_.m.p.ptr(), 0)
    }

    // 4. 从网络轮询器获取
    if netpollinited() {
        if gp := netpoll(false); gp != nil {
            return gp
        }
    }

    // 5. 从其他 P 窃取（随机选择起始位置）
    for i := 0; i < 4; i++ {
        for enum := stealOrder.start(); !enum.done(); enum.next() {
            p2 := allp[enum.position()]
            if gp := runqsteal(_g_.m.p.ptr(), p2); gp != nil {
                return gp
            }
        }
    }

    return nil
}
```

**窃取算法：**
- 随机选择一个 P 作为窃取目标
- 窃取目标 P 本地队列的一半 G
- 使用 CAS 操作保证并发安全
- 采用伪随机遍历顺序，避免冲突

**本地队列结构：**
```go
type p struct {
    runqhead uint32        // 队列头
    runqtail uint32        // 队列尾
    runq     [256]guintptr // 本地队列（环形缓冲区）
    runnext  guintptr      // 下一个优先运行的 G
}
```

**优化点：**
- runnext 字段：新创建的 G 优先放这里，提高缓存局部性
- 本地队列无锁：使用原子操作
- 窃取一半：平衡负载和窃取开销

**8. Channel 的 happens-before 保证**

**答案：**
Channel 提供了明确的内存可见性保证。

**happens-before 规则：**

1. **无缓冲 Channel：**
```go
var c = make(chan int)
var a string

func sender() {
    a = "hello"      // 1
    c <- 1           // 2
}

func receiver() {
    <-c              // 3
    print(a)         // 4: 保证输出 "hello"
}
```
**保证：** 发送操作 (2) happens-before 接收操作 (3)
因此：(1) happens-before (4)，接收方能看到发送方的修改

2. **有缓冲 Channel：**
```go
var c = make(chan int, 1)
var a string

func sender() {
    a = "hello"      // 1
    c <- 1           // 2
}

func receiver() {
    <-c              // 3
    print(a)         // 4
}
```
**保证：** 第 k 个接收 happens-before 第 k+C 个发送完成（C 是容量）

3. **Channel 关闭：**
```go
var c = make(chan int)
var a string

func sender() {
    a = "hello"      // 1
    close(c)         // 2
}

func receiver() {
    <-c              // 3: 收到零值
    print(a)         // 4: 保证输出 "hello"
}
```
**保证：** close(c) happens-before 从 c 接收到零值

**底层实现：**
- Channel 操作使用互斥锁保护
- 锁的获取和释放提供内存屏障
- 保证跨 goroutine 的内存可见性

**9. sync.Map 的实现原理与适用场景**

**答案：**
sync.Map 是为特定场景优化的并发安全 map。

**核心设计：**
```go
type Map struct {
    mu     Mutex
    read   atomic.Value  // readOnly 结构
    dirty  map[any]*entry
    misses int
}

type readOnly struct {
    m       map[any]*entry
    amended bool  // dirty 中有 read 没有的 key
}

type entry struct {
    p unsafe.Pointer  // *interface{}
}
```

**读写分离策略：**

1. **读操作（无锁快路径）：**
```go
func (m *Map) Load(key any) (value any, ok bool) {
    // 1. 先从 read map 读取（无锁）
    read := m.loadReadOnly()
    e, ok := read.m[key]

    // 2. read 中没有且 dirty 有新数据
    if !ok && read.amended {
        m.mu.Lock()
        // 双重检查
        read = m.loadReadOnly()
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            m.missLocked()  // 记录 miss
        }
        m.mu.Unlock()
    }

    if !ok {
        return nil, false
    }
    return e.load()
}
```

2. **写操作：**
```go
func (m *Map) Store(key, value any) {
    // 1. 如果 key 在 read 中，尝试直接更新
    read := m.loadReadOnly()
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 2. 需要操作 dirty map
    m.mu.Lock()
    read = m.loadReadOnly()

    if e, ok := read.m[key]; ok {
        // key 在 read 中但被标记删除
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // key 在 dirty 中
        e.storeLocked(&value)
    } else {
        // 新 key
        if !read.amended {
            // dirty 为空，从 read 复制
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}
```

3. **提升机制：**
```go
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    // miss 次数达到阈值，提升 dirty 到 read
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

**适用场景：**
- ✅ 读多写少（读写比 > 10:1）
- ✅ key 集合相对稳定
- ✅ 多个 goroutine 读写不同的 key
- ❌ 频繁增删 key
- ❌ 读写比例接近
- ❌ 需要遍历所有 key

**性能对比：**
```go
// 场景1：读多写少 - sync.Map 优势明显
// sync.Map:     100ns/op
// map+RWMutex:  500ns/op

// 场景2：频繁写入 - map+RWMutex 更好
// sync.Map:     1000ns/op
// map+RWMutex:  300ns/op
```

**10. atomic 包的内存模型与使用陷阱**

**答案：**

**atomic 操作的内存顺序保证：**

1. **基本保证：**
- atomic 操作对单个变量提供原子性
- 提供 happens-before 关系
- 不保证其他变量的可见性

2. **常见陷阱：**

**陷阱1：误以为 atomic 能同步其他变量**
```go
type Config struct {
    ready int32
    data  map[string]string
}

var config Config

// 错误：data 的修改对其他 goroutine 不一定可见
func updateConfig() {
    config.data = loadNewConfig()
    atomic.StoreInt32(&config.ready, 1)
}

func useConfig() {
    if atomic.LoadInt32(&config.ready) == 1 {
        // 可能看到旧的 data！
        use(config.data)
    }
}

// 正确：使用指针原子操作
type Config struct {
    data map[string]string
}

var configPtr atomic.Value

func updateConfig() {
    newConfig := &Config{data: loadNewConfig()}
    configPtr.Store(newConfig)
}

func useConfig() {
    config := configPtr.Load().(*Config)
    use(config.data)
}
```

**陷阱2：atomic.Value 的类型一致性**
```go
var v atomic.Value

v.Store(42)
v.Store("hello")  // panic: 类型必须一致
```

**陷阱3：false sharing 导致性能下降**
```go
// 错误：多个 atomic 变量在同一缓存行
type Counter struct {
    a int64  // 0-7 字节
    b int64  // 8-15 字节
    c int64  // 16-23 字节
}

// 正确：使用 padding 避免 false sharing
type Counter struct {
    a int64
    _ [7]int64  // padding 到 64 字节（缓存行大小）
    b int64
    _ [7]int64
    c int64
}
```

**陷阱4：CompareAndSwap 的 ABA 问题**
```go
// ABA 问题示例
var ptr unsafe.Pointer

// goroutine 1
old := atomic.LoadPointer(&ptr)  // 读到 A
// ... 被调度走
// goroutine 2: A -> B -> A
atomic.CompareAndSwapPointer(&ptr, old, new)  // 成功，但中间状态丢失

// 解决：使用版本号
type VersionedPtr struct {
    ptr     unsafe.Pointer
    version int64
}
```

**正确使用模式：**
```go
// 1. 单个标志位
var shutdown int32
atomic.StoreInt32(&shutdown, 1)
if atomic.LoadInt32(&shutdown) == 1 { }

// 2. 计数器
var counter int64
atomic.AddInt64(&counter, 1)

// 3. 配置热更新
var config atomic.Value
config.Store(&Config{})
cfg := config.Load().(*Config)

// 4. 自旋锁
type SpinLock struct {
    flag int32
}

func (s *SpinLock) Lock() {
    for !atomic.CompareAndSwapInt32(&s.flag, 0, 1) {
        runtime.Gosched()
    }
}

func (s *SpinLock) Unlock() {
    atomic.StoreInt32(&s.flag, 0)
}
```

## 2. 内存管理与GC

**11. Go 内存分配器的 TCMalloc 模型详解**

**答案：**
Go 的内存分配器基于 Google 的 TCMalloc 设计，采用多级缓存减少锁竞争。

**核心组件：**

1. **mspan（内存块）：**
```go
type mspan struct {
    next     *mspan     // 链表指针
    prev     *mspan
    startAddr uintptr   // 起始地址
    npages    uintptr   // 页数（1页=8KB）
    spanclass spanClass // span 类别

    // 对象分配位图
    allocBits  *gcBits  // 标记已分配对象
    gcmarkBits *gcBits  // GC 标记位图

    // 空闲对象链表
    freeindex uintptr
    nelems    uintptr   // 对象总数
}
```

2. **mcache（线程缓存）：**
```go
type mcache struct {
    // 每个 P 独占一个 mcache
    alloc [numSpanClasses]*mspan  // 67*2=134 个 span 类别

    // 微对象分配器（<16B）
    tiny       uintptr
    tinyoffset uintptr
    tinyAllocs uintptr
}
```

3. **mcentral（中心缓存）：**
```go
type mcentral struct {
    spanclass spanClass

    // 部分空闲的 span 列表
    partial [2]spanSet
    // 完全空闲的 span 列表
    full    [2]spanSet
}
```

4. **mheap（堆）：**
```go
type mheap struct {
    // 全局锁
    lock mutex

    // 页分配器
    pages pageAlloc

    // 中心缓存数组
    central [numSpanClasses]struct {
        mcentral mcentral
        pad      [cpu.CacheLinePadSize]byte
    }

    // 大对象直接分配
    arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena
}
```

**分配流程：**

```go
// 小对象分配（<32KB）
func mallocgc(size uintptr, typ *_type) unsafe.Pointer {
    // 1. 微对象（<16B 且无指针）
    if size <= maxTinySize && typ.ptrdata == 0 {
        off := c.tinyoffset
        if off+size <= maxTinySize && c.tiny != 0 {
            // 从 tiny 块分配
            x := unsafe.Pointer(c.tiny + off)
            c.tinyoffset = off + size
            return x
        }
        // tiny 块用完，分配新的
        span := c.alloc[tinySpanClass]
        v := nextFreeFast(span)
        if v == 0 {
            v = c.nextFree(tinySpanClass)
        }
        c.tiny = v
        c.tinyoffset = size
        return unsafe.Pointer(v)
    }

    // 2. 小对象（16B-32KB）
    var sizeclass uint8
    if size <= smallSizeMax-8 {
        sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
    } else {
        sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
    }

    span := c.alloc[sizeclass]
    v := nextFreeFast(span)
    if v == 0 {
        // mcache 无空闲，从 mcentral 获取
        v = c.nextFree(sizeclass)
    }
    return unsafe.Pointer(v)
}

// 大对象分配（>=32KB）
func largeAlloc(size uintptr) *mspan {
    // 直接从 mheap 分配
    npages := size >> _PageShift
    s := mheap_.alloc(npages, spanclass)
    return s
}
```

**分配策略：**
- **微对象（<16B）**：多个对象共享一个 16B 块，减少内存碎片
- **小对象（16B-32KB）**：从 mcache 的对应 sizeclass 分配，无锁
- **大对象（>=32KB）**：直接从 mheap 分配，需要加锁

**67 个 sizeclass：**
```
8B, 16B, 24B, 32B, 48B, 64B, 80B, 96B, 112B, 128B, ...
每个 sizeclass 有 noscan 和 scan 两个版本（是否包含指针）
```

**12. 三色标记 GC 算法与写屏障**

**答案：**

**三色标记法：**
- **白色**：未被扫描的对象（潜在垃圾）
- **灰色**：已被扫描但其引用未扫描的对象
- **黑色**：已被扫描且其引用也已扫描的对象

**GC 流程：**

```go
// 1. Mark Setup（STW）
func gcStart() {
    // 启动写屏障
    setGCPhase(_GCmark)

    // 标记根对象为灰色
    gcMarkRootPrepare()

    // 启动标记 worker
    startTheWorldWithSema()
}

// 2. Concurrent Mark（并发标记）
func gcDrain(gcw *gcWork) {
    for {
        // 从灰色队列取对象
        obj := gcw.tryGet()
        if obj == 0 {
            break
        }

        // 扫描对象，标记引用为灰色
        scanobject(obj, gcw)

        // 对象标记为黑色
        greyobject(obj, gcw)
    }
}

// 3. Mark Termination（STW）
func gcMarkTermination() {
    // 完成剩余标记工作
    gcMarkDone()

    // 关闭写屏障
    setGCPhase(_GCoff)

    // 清扫准备
    gcSweep()
}

// 4. Concurrent Sweep（并发清扫）
func sweep() {
    for s := range mheap_.allspans {
        if s.sweepgen == sweepgen-2 {
            // 清扫 span
            s.sweep(false)
        }
    }
}
```

**写屏障（Write Barrier）：**

Go 使用混合写屏障（Hybrid Write Barrier），结合了 Dijkstra 插入屏障和 Yuasa 删除屏障。

```go
// 写屏障伪代码
func writePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    // 1. Yuasa 删除屏障：标记旧值
    shade(*slot)

    // 2. Dijkstra 插入屏障：标记新值
    shade(ptr)

    // 3. 执行写操作
    *slot = ptr
}

// 实际实现（汇编）
TEXT runtime·gcWriteBarrier(SB),NOSPLIT,$0
    // 检查写屏障是否启用
    MOVQ g_m(R14), R13
    MOVQ m_p(R13), R13
    MOVQ (p_wbBuf+wbBuf_next)(R13), R12

    // 记录旧值和新值
    MOVQ AX, (R12)
    MOVQ DI, 8(R12)

    // 更新 wbBuf
    ADDQ $16, R12
    MOVQ R12, (p_wbBuf+wbBuf_next)(R13)

    // 执行写操作
    MOVQ AX, (DI)
    RET
```

**混合写屏障的优势：**
- 无需 STW 重新扫描栈
- 保证强三色不变性
- 性能开销小（~4%）

**GC 触发条件：**
```go
// 1. 堆内存达到阈值
if memstats.heap_live >= memstats.gc_trigger {
    gcStart()
}

// 2. 定时触发（2分钟）
if forcegcperiod > 0 && now-memstats.last_gc_nanotime > forcegcperiod {
    gcStart()
}

// 3. 手动触发
runtime.GC()
```

**GC Pacer（调步器）：**
```go
// 计算下次 GC 触发的堆大小
func gcSetTriggerRatio(triggerRatio float64) {
    // 目标：GC 完成时堆大小 = 当前存活 * (1 + GOGC/100)
    // GOGC 默认 100，即堆翻倍时触发 GC
    memstats.gc_trigger = uint64(float64(memstats.heap_marked) * (1 + triggerRatio))
}
```

**13. 逃逸分析的深层原理**

**答案：**

逃逸分析决定变量分配在栈还是堆，影响 GC 压力和性能。

**逃逸场景：**

1. **指针逃逸：**
```go
// 逃逸：返回局部变量指针
func escape1() *int {
    x := 42
    return &x  // x 逃逸到堆
}

// 不逃逸：不返回指针
func noEscape1() int {
    x := 42
    return x  // x 在栈上
}
```

2. **interface{} 逃逸：**
```go
// 逃逸：赋值给 interface{}
func escape2() {
    x := 42
    fmt.Println(x)  // x 逃逸（fmt.Println 参数是 interface{}）
}

// 不逃逸：类型确定
func noEscape2() {
    x := 42
    _ = x
}
```

3. **闭包捕获：**
```go
// 逃逸：闭包捕获变量
func escape3() func() int {
    x := 42
    return func() int {
        return x  // x 逃逸
    }
}

// 不逃逸：闭包不逃逸
func noEscape3() {
    x := 42
    func() {
        _ = x  // x 不逃逸
    }()
}
```

4. **切片/map 动态增长：**
```go
// 逃逸：大小不确定
func escape4(n int) []int {
    s := make([]int, n)  // 逃逸
    return s
}

// 不逃逸：大小确定且小
func noEscape4() []int {
    s := make([]int, 10)  // 可能不逃逸
    return s
}
```

5. **发送到 channel：**
```go
// 逃逸：发送指针到 channel
func escape5(ch chan *int) {
    x := 42
    ch <- &x  // x 逃逸
}
```

**逃逸分析算法：**

```go
// 编译器逃逸分析流程
func escapeAnalysis(fn *Func) {
    // 1. 构建数据流图
    buildDataFlowGraph(fn)

    // 2. 标记逃逸节点
    for _, n := range fn.nodes {
        if n.escapes() {
            markEscape(n)
        }
    }

    // 3. 传播逃逸信息
    propagateEscape(fn)

    // 4. 决定分配位置
    for _, n := range fn.nodes {
        if n.isEscape {
            n.allocOnHeap()
        } else {
            n.allocOnStack()
        }
    }
}
```

**查看逃逸分析：**
```bash
# 编译时查看逃逸分析
go build -gcflags="-m -m" main.go

# 输出示例：
# ./main.go:5:2: x escapes to heap
# ./main.go:5:2:   flow: ~r0 = &x
# ./main.go:5:2:     from &x (address-of) at ./main.go:6:9
# ./main.go:5:2:     from return &x (return) at ./main.go:6:2
```

**优化技巧：**

```go
// 1. 避免返回指针
// 差
func bad() *Result {
    r := Result{}
    return &r  // 逃逸
}

// 好
func good() Result {
    return Result{}  // 不逃逸
}

// 2. 使用 sync.Pool 复用对象
var pool = sync.Pool{
    New: func() interface{} {
        return &Buffer{}
    },
}

func process() {
    buf := pool.Get().(*Buffer)
    defer pool.Put(buf)
    // 使用 buf
}

// 3. 预分配切片
// 差
func bad(n int) []int {
    var s []int
    for i := 0; i < n; i++ {
        s = append(s, i)  // 多次扩容，逃逸
    }
    return s
}

// 好
func good(n int) []int {
    s := make([]int, 0, n)  // 预分配，减少逃逸
    for i := 0; i < n; i++ {
        s = append(s, i)
    }
    return s
}

// 4. 避免 interface{} 装箱
// 差
func bad(v int) {
    fmt.Println(v)  // v 逃逸到堆
}

// 好
func good(v int) {
    if debug {
        fmt.Println(v)  // 条件编译可能优化
    }
}
```

**14. 内存对齐与 padding**

**答案：**

内存对齐影响结构体大小和访问性能。

**对齐规则：**
```go
// 基本类型对齐
bool, int8, uint8:     1 字节对齐
int16, uint16:         2 字节对齐
int32, uint32, float32: 4 字节对齐
int64, uint64, float64: 8 字节对齐
指针:                   8 字节对齐（64位系统）

// 结构体对齐：取最大字段的对齐值
```

**示例：**

```go
// 未优化：24 字节
type Bad struct {
    a bool   // 1 字节 + 7 字节 padding
    b int64  // 8 字节
    c bool   // 1 字节 + 7 字节 padding
}

// 优化后：16 字节
type Good struct {
    b int64  // 8 字节
    a bool   // 1 字节
    c bool   // 1 字节 + 6 字节 padding
}

// 验证大小
fmt.Println(unsafe.Sizeof(Bad{}))   // 24
fmt.Println(unsafe.Sizeof(Good{}))  // 16
```

**查看内存布局：**
```go
type Example struct {
    a int8   // offset 0, size 1
    b int64  // offset 8, size 8
    c int16  // offset 16, size 2
}

func printLayout() {
    fmt.Printf("Size: %d\n", unsafe.Sizeof(Example{}))
    fmt.Printf("a offset: %d\n", unsafe.Offsetof(Example{}.a))
    fmt.Printf("b offset: %d\n", unsafe.Offsetof(Example{}.b))
    fmt.Printf("c offset: %d\n", unsafe.Offsetof(Example{}.c))
}
// 输出：
// Size: 24
// a offset: 0
// b offset: 8
// c offset: 16
```

**False Sharing 问题：**

```go
// 问题：多个 goroutine 修改相邻字段，导致缓存行失效
type Counter struct {
    a int64  // 0-7 字节
    b int64  // 8-15 字节
}

var counter Counter

func worker1() {
    for {
        atomic.AddInt64(&counter.a, 1)  // 修改 a
    }
}

func worker2() {
    for {
        atomic.AddInt64(&counter.b, 1)  // 修改 b，导致 CPU 缓存行失效
    }
}

// 解决：使用 padding 分离到不同缓存行
type Counter struct {
    a int64
    _ [7]int64  // padding 56 字节
    b int64
    _ [7]int64  // padding 56 字节
}
```

**15. sync.Pool 的实现与最佳实践**

**答案：**

sync.Pool 用于复用临时对象，减少 GC 压力。

**核心结构：**
```go
type Pool struct {
    noCopy noCopy

    // 每个 P 的本地池
    local     unsafe.Pointer  // [P]poolLocal
    localSize uintptr

    // 受害者缓存（上一轮 GC 的对象）
    victim     unsafe.Pointer
    victimSize uintptr

    // 对象构造函数
    New func() interface{}
}

type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})]byte  // 避免 false sharing
}

type poolLocalInternal struct {
    private interface{}   // 私有对象（无锁）
    shared  poolChain     // 共享队列（lock-free）
}
```

**Get/Put 流程：**

```go
func (p *Pool) Get() interface{} {
    // 1. 尝试从当前 P 的 private 获取
    l := p.pin()
    x := l.private
    l.private = nil
    if x != nil {
        runtime_procUnpin()
        return x
    }

    // 2. 从当前 P 的 shared 队列获取
    x = l.shared.popHead()
    if x != nil {
        runtime_procUnpin()
        return x
    }
    runtime_procUnpin()

    // 3. 从其他 P 窃取
    x = p.getSlow()
    if x != nil {
        return x
    }

    // 4. 从 victim 缓存获取
    // ...

    // 5. 调用 New 创建
    if p.New != nil {
        return p.New()
    }
    return nil
}

func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }

    // 1. 尝试放入当前 P 的 private
    l := p.pin()
    if l.private == nil {
        l.private = x
        x = nil
    }
    runtime_procUnpin()

    // 2. 放入 shared 队列
    if x != nil {
        l.shared.pushHead(x)
    }
}
```

**GC 清理机制：**
```go
func poolCleanup() {
    // 每次 GC 时调用
    for _, p := range allPools {
        // victim 缓存移到 victim
        p.victim = p.local
        p.victimSize = p.localSize

        // 清空 local
        p.local = nil
        p.localSize = 0
    }

    // 清空旧的 victim
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }
}
```

**最佳实践：**

```go
// 1. 正确使用场景
// ✅ 临时对象，生命周期短
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset()  // 重要：重置状态
    defer bufferPool.Put(buf)

    buf.Write(data)
    // 处理 buf
}

// ❌ 不适合：长期持有的对象
var connPool = sync.Pool{  // 错误！连接应该用连接池
    New: func() interface{} {
        return openConnection()
    },
}

// 2. 重置对象状态
type Object struct {
    data []byte
    refs int
}

var objPool = sync.Pool{
    New: func() interface{} {
        return &Object{
            data: make([]byte, 0, 1024),
        }
    },
}

func getObject() *Object {
    obj := objPool.Get().(*Object)
    obj.data = obj.data[:0]  // 重置切片
    obj.refs = 0             // 重置字段
    return obj
}

// 3. 避免内存泄漏
func putObject(obj *Object) {
    // 清空大对象引用
    if cap(obj.data) > 64*1024 {
        obj.data = nil  // 避免池中保留大内存
    }
    objPool.Put(obj)
}

// 4. 性能测试
func BenchmarkWithPool(b *testing.B) {
    for i := 0; i < b.N; i++ {
        buf := bufferPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.WriteString("test")
        bufferPool.Put(buf)
    }
}

func BenchmarkWithoutPool(b *testing.B) {
    for i := 0; i < b.N; i++ {
        buf := new(bytes.Buffer)
        buf.WriteString("test")
    }
}
// 结果：WithPool 快 3-5 倍，分配次数减少 90%
```

**注意事项：**
- Pool 中的对象可能随时被 GC 回收
- 不要依赖 Pool 做对象缓存（用 LRU 等）
- Put 前必须重置对象状态
- 不要 Put 大对象（>64KB）
- 适合高频创建销毁的小对象

## 3. 数据结构与类型系统

**19-26题：slice、map、interface、类型断言、defer、panic/recover、值传递**

见主文件详细答案（题19-26）

## 4. 网络编程

**27-33题：net/http、HTTP服务器、gRPC、WebSocket、TCP粘包、IO模型、连接池**

见主文件详细答案（题27-33）

## 5. 性能优化

**34-39题：pprof、性能瓶颈、CPU优化、内存优化、benchmark、内联优化**

见主文件详细答案（题34-39）
