# 迅雷网盘特性与关键技术分析

## 核心特性

### 1. **P2P加速下载**
- 利用P2P技术从多个节点同时下载
- 边下边播技术
- 充分利用用户带宽资源

### 2. **离线下载**
- 服务器端下载资源
- 用户无需在线等待
- 支持BT、磁力链接等多种协议

### 3. **秒传功能**
- 文件hash去重
- 已存在文件直接关联
- 节省上传时间和存储空间

### 4. **大文件分片上传**
- 断点续传
- 并发上传
- 提高上传成功率

### 5. **智能限速与加速**
- 会员加速通道
- 动态带宽分配
- QoS服务质量保证

---

## 关键技术总结

### 1. **分布式存储**
- **对象存储(OSS)**：海量文件存储
- **分片存储**：大文件切分存储
- **冗余备份**：多副本保证可靠性
- **一致性Hash**：数据分布与负载均衡

```
文件存储架构：
Client → Upload Service → Object Storage (OSS)
                        ↓
                   Metadata DB
```

### 2. **P2P技术**
- **BitTorrent协议**：分布式文件传输
- **DHT分布式哈希表**：无中心化节点发现
- **Peer节点管理**：
  - Tracker服务器协调
  - Peer连接与数据交换
  - 上传下载速率控制

```
P2P下载流程：
1. 获取种子/磁力链接
2. 连接Tracker获取Peer列表
3. 从多个Peer并发下载分片
4. 边下载边上传(做种)
```

### 3. **文件去重技术(秒传)**
- **Hash计算**：SHA-1/MD5文件指纹
- **内容寻址存储(CAS)**：相同内容只存一份
- **秒传实现原理**：
  ```
  1. 客户端计算文件hash
  2. 服务端查询hash是否存在
  3. 存在则直接创建用户文件引用
  4. 不存在则执行真实上传
  ```

```sql
-- 文件存储表设计
CREATE TABLE files (
    file_hash VARCHAR(64) PRIMARY KEY,
    file_size BIGINT,
    storage_path VARCHAR(255),
    ref_count INT,  -- 引用计数
    created_at TIMESTAMP
);

CREATE TABLE user_files (
    user_id BIGINT,
    file_hash VARCHAR(64),
    file_name VARCHAR(255),
    folder_path VARCHAR(500),
    created_at TIMESTAMP,
    INDEX(user_id, folder_path)
);
```

### 4. **分片上传/下载**
- **分片策略**：通常2-4MB/片
- **并发传输**：多线程同时上传/下载
- **断点续传**：
  ```
  1. 记录已完成分片
  2. 失败后从断点继续
  3. 所有分片完成后合并
  4. 校验文件完整性
  ```

```javascript
// 分片上传伪代码
const CHUNK_SIZE = 2 * 1024 * 1024; // 2MB
const chunks = Math.ceil(file.size / CHUNK_SIZE);

for (let i = 0; i < chunks; i++) {
    const chunk = file.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE);
    await uploadChunk(chunk, i, fileHash);
}
await mergeChunks(fileHash, chunks);
```

### 5. **CDN加速**
- **边缘节点缓存**：热点文件就近访问
- **智能调度**：根据地理位置分配节点
- **预加载机制**：热门资源提前缓存
- **回源策略**：缓存未命中时回源获取

### 6. **消息队列**
- **离线下载任务队列**：异步处理下载任务
- **任务调度**：优先级队列管理
- **技术选型**：Redis/RabbitMQ/Kafka

```
离线下载流程：
User Request → API Gateway → Task Queue → Download Worker
                                              ↓
                                         OSS Storage
                                              ↓
                                         Notify User
```

### 7. **限流与QoS**
- **令牌桶算法**：平滑限流
- **会员分级**：
  - 普通用户：限速下载
  - 会员用户：高速通道
  - 超级会员：极速通道
- **动态带宽分配**：根据网络状况调整

```python
# 令牌桶限流伪代码
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate  # 令牌生成速率
        self.capacity = capacity  # 桶容量
        self.tokens = capacity

    def consume(self, tokens):
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False
```

### 8. **缓存策略**
- **多级缓存架构**：
  ```
  Client Cache (本地)
       ↓
  Redis Cache (热点元数据)
       ↓
  CDN Cache (文件内容)
       ↓
  OSS Storage (持久化)
  ```
- **缓存更新策略**：LRU/LFU
- **缓存预热**：热门文件提前加载

### 9. **安全技术**
- **文件加密**：AES加密存储
- **传输加密**：TLS/SSL
- **访问控制**：
  - 用户身份认证
  - 文件权限管理
  - 分享链接加密
- **防盗链**：Referer检查、Token验证

### 10. **负载均衡**
- **接入层**：Nginx/LVS
- **应用层**：服务器集群
- **调度算法**：
  - 轮询(Round Robin)
  - 最少连接(Least Connection)
  - 一致性Hash

### 11. **数据库设计**
```sql
-- 用户表
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    vip_level INT,
    storage_quota BIGINT,
    used_storage BIGINT
);

-- 离线任务表
CREATE TABLE offline_tasks (
    task_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    url TEXT,
    status ENUM('pending', 'downloading', 'completed', 'failed'),
    progress INT,
    created_at TIMESTAMP,
    INDEX(user_id, status)
);

-- 分享表
CREATE TABLE shares (
    share_id VARCHAR(32) PRIMARY KEY,
    user_id BIGINT,
    file_hash VARCHAR(64),
    password VARCHAR(10),
    expire_time TIMESTAMP,
    download_count INT
);
```

### 12. **监控与容灾**
- **实时监控**：
  - 服务健康检查
  - 性能指标监控
  - 异常告警
- **容灾备份**：
  - 多地域部署
  - 数据多副本
  - 故障自动切换
- **降级策略**：高峰期限制非核心功能

---

## 技术架构图

```
                    ┌─────────────┐
                    │   用户端    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  CDN/负载均衡 │
                    └──────┬──────┘
                           │
        ┏━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━┓
        ┃                                      ┃
   ┌────▼────┐                          ┌─────▼─────┐
   │ API网关  │                          │  P2P节点  │
   └────┬────┘                          └───────────┘
        │
   ┌────▼────────────────────┐
   │    应用服务集群          │
   │ ┌────────┬──────┬─────┐ │
   │ │上传服务│下载  │分享 │ │
   │ └────────┴──────┴─────┘ │
   └────┬────────────────────┘
        │
   ┌────▼──────────────────────┐
   │   中间件层                 │
   │ ┌──────┬────────┬───────┐ │
   │ │Redis │消息队列│缓存   │ │
   │ └──────┴────────┴───────┘ │
   └────┬──────────────────────┘
        │
   ┌────▼──────────────────────┐
   │   存储层                   │
   │ ┌──────┬────────┬───────┐ │
   │ │MySQL │对象存储│备份   │ │
   │ └──────┴────────┴───────┘ │
   └───────────────────────────┘
```

---

## 性能优化要点

1. **上传优化**：分片并发、秒传去重、断点续传
2. **下载优化**：P2P加速、CDN分发、多线程下载
3. **存储优化**：去重存储、压缩、冷热数据分离
4. **网络优化**：协议优化(HTTP/2、QUIC)、连接复用
5. **数据库优化**：索引优化、读写分离、分库分表

---

## 面试常见问题

### Q1: 秒传是如何实现的？
**答**：通过文件hash去重实现。客户端先计算文件MD5/SHA-1，发送给服务器查询，如果服务器已存在该hash的文件，直接在用户文件表中创建引用关系，无需真实上传。

### Q2: 如何保证大文件上传的可靠性？
**答**：
- 分片上传：将大文件切分成小块
- 断点续传：记录已上传分片，失败后继续
- 并发上传：多个分片同时上传
- 校验机制：每个分片MD5校验，最终文件完整性校验

### Q3: P2P下载如何提速？
**答**：
- 从多个Peer同时下载不同分片
- 稀有分片优先下载策略
- 动态调整连接数和速率
- 本地缓存已下载分片供其他用户使用

### Q4: 如何防止存储空间被滥用？
**答**：
- 用户配额限制
- 文件去重(多用户共享同一文件不重复占用空间)
- 垃圾文件清理机制
- 冷数据归档压缩

### Q5: 离线下载的实现原理？
**答**：
- 用户提交下载任务到消息队列
- 后台Worker从队列获取任务
- Worker在服务器端执行下载
- 下载完成后通知用户
- 用户可直接从服务器高速下载到本地
