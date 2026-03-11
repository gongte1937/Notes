# Load Balancer 总结

## 一、Load Balancer 是什么？

Load Balancer（负载均衡）是把请求分发到多个后端实例的流量入口层，核心目标是：

- 横向扩展（Horizontal Scaling）
- 高可用（High Availability）
- 故障转移（Failover）
- 流量治理（Traffic Management）

---

## 二、在系统架构中的位置

典型 Web 架构：

```text
Users
  ↓
DNS
  ↓
Load Balancer
  ↓
Application Servers
  ↓
Cache / Database
```

> 负载均衡通常是可扩展系统的第一层流量控制点。

---

## 三、为什么需要负载均衡？

单机服务的典型问题：

- 单点故障（SPOF）
- 吞吐量有限
- 版本发布风险高（难做滚动升级）
- 无法平滑扩容

引入 Load Balancer 后可以：

- 在多实例之间自动分发流量
- 配合 Auto Scaling 弹性扩容
- 故障节点自动摘除
- 让服务层实现无停机发布

---

## 四、L4 和 L7 的区别

| 维度 | L4 Load Balancer | L7 Load Balancer |
| --- | --- | --- |
| 工作层 | TCP / UDP | HTTP / HTTPS |
| 路由依据 | IP + Port | Path / Header / Cookie / Host |
| 性能 | 更高 | 略低但能力更强 |
| 典型能力 | 连接转发 | TLS 终止、内容路由、灰度发布 |
| 适用场景 | RPC、游戏、长连接网关 | Web、微服务、SaaS |

示例（L7）：

```text
/api     -> API Cluster
/static  -> CDN or Static Service
/admin   -> Internal Admin Service
```

---

## 五、常见负载均衡算法

### 1. Round Robin

按顺序轮询分发请求，简单但不感知实例负载差异。

### 2. Weighted Round Robin

按权重分流，适合异构机器（不同 CPU/内存规格混布）。

### 3. Least Connections

优先分配给当前连接数更少的实例，适合长连接、WebSocket 等场景。

### 4. Hash / Consistent Hash

按 `hash(key)`（如 userId、sessionId、client IP）路由到固定实例。

- 优点：路由稳定，可用于会话粘性
- 风险：扩缩容会带来映射变化，需要一致性哈希环来降低迁移量

---

## 六、Health Check 与故障摘除

Load Balancer 会周期性探测实例健康状态，例如：

```text
GET /health
```

若连续探测失败（超时、5xx、连接失败），实例会被临时摘除；恢复后再加入流量池。

> 健康检查 + 自动摘除是高可用的核心机制。

---

## 七、Sticky Session 与无状态设计

若会话仅存于单机内存，会出现：

```text
请求1 -> Server A（登录）
请求2 -> Server B（会话丢失）
```

常见方案：

- Sticky Session（短期可用，但会影响流量均衡）
- Session 外置（Redis 等）
- Stateless + Token（如 JWT，推荐）

---

## 八、Load Balancer 自身的高可用

Load Balancer 也可能成为单点，需要多层保护：

- Active-Standby 或 Active-Active
- 多可用区（Multi-AZ）部署
- DNS 层故障切换
- 跨地域 Global Load Balancing

---

## 九、与数据库扩展的关系

数据库也有“负载分发”思想：

- 写请求 -> Primary
- 读请求 -> Replicas

可结合 [[Database Replication]] 与 [[Database Sharding]] 理解：

- Replication 侧重读扩展与容灾
- Sharding 侧重容量与写压力拆分

---

## 十、面试回答框架（高可用 Web 系统）

1. DNS 入口接入 Load Balancer
2. 多实例部署应用层，LB 做流量分发
3. 配置健康检查与自动摘除
4. 服务无状态化（Session 外置或 Token 化）
5. 数据层做读写分离，必要时分片
6. 配合 Auto Scaling + 滚动发布
7. 多 AZ / 跨区域容灾

---

## 十一、常见误区

- “有了 LB 就一定高可用”
错。若 LB 单点或健康检查配置不当，仍会整体不可用。

- “LB 可以解决所有性能问题”
错。LB 主要解决入口分发，不解决慢 SQL、锁竞争、存储瓶颈。

- “Sticky Session 是长期最佳方案”
错。规模变大后，通常应转向无状态服务。

---

## 十二、一句话总结

> Load Balancer 是可扩展系统的流量入口，负责把可用性、弹性扩容和发布治理串成一个整体。
