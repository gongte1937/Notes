
# Database Replication 总结

## 一、Replication 是什么？

Replication（复制）是：

> 把同一份数据复制到多台数据库服务器上。

与 Sharding 不同：

- Sharding → 数据拆开（每台不同）
- Replication → 数据相同（每台一样）

---

## 二、最常见结构：Primary + Replica

```text
Primary（主库）
  ↓
Replica 1（从库）
Replica 2（从库）
Replica 3（从库）
```

---

## 三、写入流程

```sql
INSERT INTO users ...
```

1. 写入 Primary
2. Primary 记录变更日志（binlog / WAL）
3. Replica 同步日志
4. Replica 重放日志
5. 数据保持一致

---

## 四、查询流程

通常：

- 写操作 → Primary
- 读操作 → Replica

示例：

```sql
SELECT * FROM users WHERE id = 6;
```

可能被路由到某个 Replica（例如 Replica 2）。

---

## 五、Replication 解决什么问题？

### 1. 读扩展

当单机无法承载高并发读时，增加多台 Replica 分担读流量。

### 2. 高可用（HA）

Primary 宕机时，可将某个 Replica 提升为新的 Primary，业务快速恢复。

### 3. 备份保障

多副本降低数据丢失风险。

---

## 六、Replication 和 Sharding 的核心区别

| | Replication | Sharding |
| --- | --- | --- |
| 数据内容 | 每台一样 | 每台不同 |
| 主要目的 | 提高读能力、提升可用性 | 扩容数据容量 |
| 总存储量 | 增加（多份数据） | 不变（数据拆分） |
| 单表过大问题 | 不能直接解决 | 可以解决 |

---

## 七、类比：仓库模型

原来：北京一个仓库（全量库存）

Replication：

```text
北京：完整库存
上海：完整库存
深圳：完整库存
```

Sharding：

```text
北京：只存 A–M
上海：只存 N–Z
```

---

## 八、Replication 的问题

### 1. 延迟（Replication Lag）

Primary 写入到 Replica 可见存在延迟，呈现为最终一致性（Eventually Consistent）。

### 2. 写入瓶颈

写入仍集中在 Primary，Replication 不解决写放大/写瓶颈。

---

## 九、Replication 类型

### 1. 同步复制（Synchronous）

- Primary 等待 Replica 确认
- 数据更安全
- 延迟更高

### 2. 异步复制（Asynchronous）

- Primary 写入后即返回
- Replica 异步追日志
- 性能更好
- 极端情况下可能丢少量数据

---

## 十、Sharding + Replication 结合

真实生产通常是二者结合：

```text
Shard 1
  ├── Primary
  ├── Replica
  └── Replica

Shard 2
  ├── Primary
  ├── Replica
  └── Replica
```

- Sharding → 扩容量
- Replication → 扩读 & 高可用

---

## 十一、终极总结

> Index：单机查得快

> Replication：读扩展 + 容灾

> Sharding：容量扩展

---
