## 二、PHP 核心技术（9年经验）

### 1. PHP基础与高级特性

**40. PHP的生命周期（SAPI、Zend引擎）？**

**答案：**
**PHP架构：**
```
SAPI层 → Zend引擎 → 扩展层
```

**生命周期：**
1. **模块初始化（MINIT）**：
   - PHP启动时执行一次
   - 初始化扩展、注册常量

2. **请求初始化（RINIT）**：
   - 每个请求开始时执行
   - 初始化变量、符号表

3. **执行PHP脚本**：
   - 词法分析 → 语法分析 → 编译 → 执行

4. **请求结束（RSHUTDOWN）**：
   - 清理变量、关闭资源

5. **模块关闭（MSHUTDOWN）**：
   - PHP关闭时执行
   - 清理扩展

**SAPI类型：**
- CLI：命令行
- FPM：FastCGI进程管理器
- Apache模块
- CGI

**41. PHP的垃圾回收机制（引用计数、循环引用）？**

**答案：**
**引用计数：**
```php
$a = "hello";  // refcount=1
$b = $a;       // refcount=2
unset($a);     // refcount=1
unset($b);     // refcount=0, 释放内存
```

**循环引用问题：**
```php
$a = [];
$b = [];
$a['b'] = $b;
$b['a'] = $a;
unset($a, $b);  // 内存泄漏（PHP 5.2）
```

**PHP 5.3+解决方案：**
- 使用**根缓冲区**收集可能的循环引用
- 定期运行**标记-清除算法**
- 找出真正的垃圾并释放

**触发GC：**
```php
gc_collect_cycles();  // 手动触发
gc_enable();          // 启用GC
gc_disable();         // 禁用GC
```

**42. PHP 7相比PHP 5有哪些重大改进？**

**答案：**
**性能提升：**
- 速度提升2倍
- 内存使用减少50%

**核心改进：**
1. **新增类型声明：**
```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

2. **返回类型声明：**
```php
function getName(): string {
    return "John";
}
```

3. **null合并运算符：**
```php
$name = $_GET['name'] ?? 'default';
```

4. **太空船运算符：**
```php
echo 1 <=> 2;  // -1
echo 2 <=> 2;  // 0
echo 3 <=> 2;  // 1
```

5. **匿名类：**
```php
$obj = new class {
    public function hello() {
        return "Hello";
    }
};
```

6. **错误处理改进：**
```php
try {
    // code
} catch (Error $e) {
    // 捕获致命错误
}
```

**43. PHP的命名空间和自动加载机制？**

**答案：**
**命名空间：**
```php
namespace App\Controllers;

class UserController {
}

// 使用
use App\Controllers\UserController;
$controller = new UserController();
```

**自动加载（PSR-4）：**
```php
spl_autoload_register(function ($class) {
    $prefix = 'App\\';
    $base_dir = __DIR__ . '/src/';

    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        return;
    }

    $relative_class = substr($class, $len);
    $file = $base_dir . str_replace('\\', '/', $relative_class) . '.php';

    if (file_exists($file)) {
        require $file;
    }
});
```

**Composer自动加载：**
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

**44. trait的作用和使用场景？与继承的区别？**

**答案：**
**trait定义：**
代码复用机制，解决单继承限制。

**使用示例：**
```php
trait Logger {
    public function log($message) {
        echo "[LOG] $message\n";
    }
}

trait Cache {
    public function cache($key, $value) {
        // 缓存逻辑
    }
}

class User {
    use Logger, Cache;

    public function save() {
        $this->log("Saving user");
        $this->cache("user:1", $this);
    }
}
```

**冲突解决：**
```php
trait A {
    public function hello() {
        echo "A";
    }
}

trait B {
    public function hello() {
        echo "B";
    }
}

class C {
    use A, B {
        B::hello insteadof A;  // 使用B的hello
        A::hello as helloA;    // A的hello重命名
    }
}
```

**与继承的区别：**
| 特性 | trait | 继承 |
|------|-------|------|
| 数量 | 多个 | 单个 |
| 优先级 | 高于父类 | 低于子类 |
| 用途 | 代码复用 | is-a关系 |

**45. 魔术方法有哪些？各自的作用？**

**答案：**
```php
class Magic {
    // 构造/析构
    public function __construct() {}
    public function __destruct() {}

    // 属性访问
    public function __get($name) {
        return $this->data[$name] ?? null;
    }

    public function __set($name, $value) {
        $this->data[$name] = $value;
    }

    public function __isset($name) {
        return isset($this->data[$name]);
    }

    public function __unset($name) {
        unset($this->data[$name]);
    }

    // 方法调用
    public function __call($name, $args) {
        echo "调用不存在的方法: $name";
    }

    public static function __callStatic($name, $args) {
        echo "调用不存在的静态方法: $name";
    }

    // 对象转换
    public function __toString() {
        return "Magic Object";
    }

    public function __invoke() {
        echo "对象当作函数调用";
    }

    // 序列化
    public function __sleep() {
        return ['data'];  // 返回要序列化的属性
    }

    public function __wakeup() {
        // 反序列化后的初始化
    }

    // 克隆
    public function __clone() {
        // 克隆时的处理
    }
}
```

**46. 闭包和匿名函数的区别？use关键字的作用？**

**答案：**
**匿名函数：**
```php
$greet = function($name) {
    return "Hello, $name";
};
echo $greet("John");
```

**闭包（使用use）：**
```php
$message = "Hello";
$greet = function($name) use ($message) {
    return "$message, $name";
};
echo $greet("John");  // Hello, John
```

**引用传递：**
```php
$count = 0;
$counter = function() use (&$count) {
    $count++;
};
$counter();
$counter();
echo $count;  // 2
```

**区别：**
- 匿名函数：没有名字的函数
- 闭包：捕获外部变量的匿名函数

**47. 生成器（Generator）的原理和使用场景？**

**答案：**
**原理：**
使用yield关键字，按需生成值，节省内存。

**示例：**
```php
// 普通方式（占用大量内存）
function getRange($max) {
    $array = [];
    for ($i = 0; $i < $max; $i++) {
        $array[] = $i;
    }
    return $array;
}

// 生成器方式（节省内存）
function getRange($max) {
    for ($i = 0; $i < $max; $i++) {
        yield $i;
    }
}

foreach (getRange(1000000) as $num) {
    echo $num;
}
```

**发送值到生成器：**
```php
function printer() {
    while (true) {
        $value = yield;
        echo $value . "\n";
    }
}

$gen = printer();
$gen->send("Hello");
$gen->send("World");
```

**使用场景：**
- 处理大文件
- 无限序列
- 协程实现

### 2. Laravel框架

**48. Laravel的请求生命周期？**

**答案：**
```
1. public/index.php（入口）
   ↓
2. 加载Composer自动加载
   ↓
3. 创建Application实例
   ↓
4. 创建HTTP Kernel
   ↓
5. 处理请求
   ├─ 启动服务提供者
   ├─ 执行中间件
   ├─ 路由匹配
   ├─ 执行控制器
   └─ 返回响应
   ↓
6. 发送响应
   ↓
7. 终止中间件
```

**49. 服务容器（IoC容器）的实现原理？**

**答案：**
**核心概念：**
- 依赖注入容器
- 自动解析依赖

**绑定：**
```php
// 绑定接口到实现
app()->bind(UserRepositoryInterface::class, UserRepository::class);

// 单例绑定
app()->singleton(Cache::class, function($app) {
    return new RedisCache();
});
```

**解析：**
```php
// 自动解析依赖
class UserController {
    public function __construct(UserRepository $repo) {
        $this->repo = $repo;
    }
}

// Laravel自动注入UserRepository实例
$controller = app(UserController::class);
```

**50. 依赖注入的优势？如何实现？**

**答案：**
**优势：**
- 降低耦合
- 易于测试
- 提高可维护性

**实现方式：**
```php
// 1. 构造函数注入
class UserService {
    public function __construct(
        private UserRepository $repo,
        private Cache $cache
    ) {}
}

// 2. 方法注入
public function store(Request $request, UserRepository $repo) {
    $repo->create($request->all());
}

// 3. 属性注入（不推荐）
class UserService {
    #[Inject]
    public UserRepository $repo;
}
```

**51. 中间件的执行流程？如何自定义中间件？**

**答案：**
**执行流程：**
```
请求 → 中间件1前 → 中间件2前 → 控制器 → 中间件2后 → 中间件1后 → 响应
```

**自定义中间件：**
```php
class CheckAge {
    public function handle($request, Closure $next) {
        if ($request->age < 18) {
            return redirect('home');
        }

        $response = $next($request);  // 继续处理

        // 响应后处理
        $response->header('X-Custom', 'value');

        return $response;
    }
}
```

**注册：**
```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    'check.age' => CheckAge::class,
];

// 使用
Route::get('/admin', function() {
})->middleware('check.age');
```

**52. Eloquent ORM的N+1查询问题如何解决？**

**答案：**
**N+1问题：**
```php
// 产生N+1查询
$users = User::all();  // 1次查询
foreach ($users as $user) {
    echo $user->posts;  // N次查询
}
```

**解决方案：**
```php
// 1. 预加载（Eager Loading）
$users = User::with('posts')->get();  // 2次查询

// 2. 延迟预加载
$users = User::all();
$users->load('posts');

// 3. 嵌套预加载
$users = User::with('posts.comments')->get();

// 4. 条件预加载
$users = User::with(['posts' => function($query) {
    $query->where('published', true);
}])->get();
```

**53. Laravel的队列系统如何实现？支持哪些驱动？**

**答案：**
**支持的驱动：**
- Redis
- Database
- Beanstalkd
- SQS（AWS）
- Sync（同步，用于测试）

**创建任务：**
```php
class SendEmail implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle() {
        // 发送邮件逻辑
    }
}
```

**分发任务：**
```php
// 立即分发
SendEmail::dispatch($user);

// 延迟分发
SendEmail::dispatch($user)->delay(now()->addMinutes(10));

// 指定队列
SendEmail::dispatch($user)->onQueue('emails');
```

**运行队列：**
```bash
php artisan queue:work
php artisan queue:work --queue=high,default
```

**54. 事件和监听器的使用场景？**

**答案：**
**定义事件：**
```php
class UserRegistered {
    public function __construct(public User $user) {}
}
```

**定义监听器：**
```php
class SendWelcomeEmail {
    public function handle(UserRegistered $event) {
        Mail::to($event->user)->send(new WelcomeEmail());
    }
}
```

**注册：**
```php
// EventServiceProvider
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmail::class,
        LogUserRegistration::class,
    ],
];
```

**触发事件：**
```php
event(new UserRegistered($user));
```

**使用场景：**
- 解耦业务逻辑
- 异步处理
- 日志记录
- 通知发送

**55. Laravel的缓存策略？如何实现缓存标签？**

**答案：**
**缓存操作：**
```php
// 存储
Cache::put('key', 'value', 3600);

// 获取
$value = Cache::get('key', 'default');

// 永久存储
Cache::forever('key', 'value');

// 记忆（不存在时执行回调）
$value = Cache::remember('users', 3600, function() {
    return DB::table('users')->get();
});
```

**缓存标签：**
```php
// 存储带标签的缓存
Cache::tags(['people', 'artists'])->put('John', $john, 3600);
Cache::tags(['people', 'authors'])->put('Anne', $anne, 3600);

// 获取
$john = Cache::tags(['people', 'artists'])->get('John');

// 清除标签
Cache::tags(['people'])->flush();  // 清除所有people标签的缓存
```

**注意：**
- 只有Redis和Memcached支持标签
- Database和File驱动不支持

### 3. 性能优化

**56. PHP-FPM的工作原理？如何调优？**

**答案：**
**工作原理：**
```
Nginx → FastCGI → PHP-FPM Master → Worker进程池
```

**进程管理模式：**
1. **static**：固定数量的子进程
2. **dynamic**：动态调整子进程数量
3. **ondemand**：按需启动

**配置优化：**
```ini
; 进程管理
pm = dynamic
pm.max_children = 50        ; 最大子进程数
pm.start_servers = 10       ; 启动时子进程数
pm.min_spare_servers = 5    ; 最小空闲进程
pm.max_spare_servers = 15   ; 最大空闲进程
pm.max_requests = 500       ; 每个进程处理请求数后重启

; 超时
request_terminate_timeout = 30

; 慢日志
slowlog = /var/log/php-fpm-slow.log
request_slowlog_timeout = 5s
```

**调优建议：**
- 根据内存计算max_children：`总内存 / 单进程内存`
- 监控进程状态：`/status`
- 启用慢日志排查性能问题

**57. OPcache的作用？如何配置？**

**答案：**
**作用：**
- 缓存编译后的opcode
- 避免重复编译
- 提升性能2-3倍

**配置：**
```ini
opcache.enable=1
opcache.memory_consumption=128      ; 内存大小
opcache.interned_strings_buffer=8  ; 字符串缓存
opcache.max_accelerated_files=4000 ; 最大缓存文件数
opcache.revalidate_freq=60         ; 检查更新频率（秒）
opcache.fast_shutdown=1            ; 快速关闭
opcache.enable_cli=0               ; CLI模式禁用
```

**生产环境优化：**
```ini
opcache.validate_timestamps=0  ; 禁用时间戳验证（需手动清除缓存）
opcache.save_comments=0        ; 不保存注释
```

**清除缓存：**
```php
opcache_reset();
```

**58. 如何优化PHP的内存使用？**

**答案：**
**优化策略：**
```php
// 1. 及时释放变量
unset($largeArray);

// 2. 使用生成器处理大数据
function readLargeFile($file) {
    $handle = fopen($file, 'r');
    while (!feof($handle)) {
        yield fgets($handle);
    }
    fclose($handle);
}

// 3. 分批处理
User::chunk(1000, function($users) {
    foreach ($users as $user) {
        // 处理
    }
});

// 4. 使用引用传递大对象
function process(&$largeObject) {
    // 避免复制
}

// 5. 限制内存
ini_set('memory_limit', '128M');
```

**59. 如何处理PHP的慢查询问题？**

**答案：**
见MySQL慢查询优化（第65题）。

**60. Composer的自动加载优化策略？**

**答案：**
**优化命令：**
```bash
# 生成优化的自动加载文件
composer dump-autoload --optimize

# 或在安装时优化
composer install --optimize-autoloader --no-dev

# 类映射权威（最快）
composer dump-autoload --classmap-authoritative
```

**原理：**
- 生成类映射表
- 避免文件系统查找
- 提升自动加载速度50%+

**配置：**
```json
{
    "config": {
        "optimize-autoloader": true,
        "classmap-authoritative": true
    }
}
```

---

