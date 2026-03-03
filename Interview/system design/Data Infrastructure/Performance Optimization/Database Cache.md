

 # Database Cache 总结

## 一、Cache 是什么？

Cache 是：

> 把“高频访问数据”放到更快的存储中（通常是内存）。

核心目标：

- 减少数据库访问
- 减少磁盘 I/O
- 提高响应速度
- 扛住高并发

---

## 二、为什么需要 Cache？

假设：

- 每秒 10 万次请求
- 每次都查数据库

结果：数据库容易被打爆。

加 Cache 之后：

```text
Client
  ↓
Cache（命中）
  ↓
直接返回
```

> 数据库压力明显下降。

---

## 三、常见缓存系统

代表系统：

- Redis
- Memcached

特点：

- 基于内存
- 读写极快（微秒级）
- 支持 key-value 存储

---

## 四、常见缓存模式

### 1. Cache Aside（最常用）

读流程：

1. 先查 Cache
2. 命中 → 直接返回
3. 未命中 → 查 DB
4. 将结果写入 Cache

写流程：

1. 更新 DB
2. 删除 Cache（让下次读重建）

优点：简单、易控制；缺点：可能短暂不一致。

---

### 2. Write Through

写流程：

1. 写 Cache
2. Cache 同步写 DB

优点：一致性强；缺点：写延迟高。

---

### 3. Write Back（不常用）

写流程：

1. 写 Cache
2. 定期批量刷 DB

优点：写性能高；缺点：异常情况下可能丢数据。

---

## 五、缓存常见问题（面试高频）

### 1. 缓存击穿（Cache Breakdown）

热点 Key 过期瞬间，大量请求同时打到 DB。

解决：加锁（mutex）、热点永不过期、逻辑过期。

### 2. 缓存穿透（Cache Penetration）

查询不存在的数据，每次都查 DB。

解决：缓存空值、布隆过滤器。

### 3. 缓存雪崩（Cache Avalanche）

大量 Key 同时过期导致 DB 被打爆。

解决：过期时间随机化、分批过期、多级缓存。

---

## 六、Cache 在系统设计中的位置

典型架构：

```text
Client
  ↓
API Server
  ↓
Cache
  ↓
Database
```

原则：

> 优先查 Cache，尽量避免请求直接打到 DB。

---

## 七、什么时候必须用 Cache？

- 高并发读（Feed / 热门内容）
- 排行榜
- 商品详情页
- 秒杀系统
    
- Session 存储
    

---

# 八、Cache + Sharding + Replication

真实生产架构：

- Cache 扛流量
    
- Sharding 扛数据规模
    
- Replication 扛读 & 高可用
    

三者配合。

---

# 九、缓存一致性问题

常见问题：

- DB 更新了，Cache 还是旧数据
    

解决方案：

- 先更新 DB，再删除 Cache
    
- 延迟双删
    
- 消息队列同步
    

---

# 十、一句话总结

Cache =

> 用内存换数据库压力，用空间换性能。

在高并发系统中：

真正抗住流量的不是数据库，是缓存。
