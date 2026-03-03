
## 一、什么是 Normalization？

Normalization（规范化）是一种数据库设计方法，用来：

- 减少数据冗余
- 避免数据不一致
- 提高数据结构清晰度
- 消除更新异常（Update Anomaly）

核心思想：

> 将重复数据拆分成独立的表，并通过主键与外键建立关联。

---

## 二、为什么需要 Normalization？

### 未规范化示例（❌）

| order_id | user_name | user_email  | product_name |
| --- | --- | --- | --- |
| 1 | Alice | a@gmail.com | iPhone |
| 2 | Alice | a@gmail.com | AirPods |
| 3 | Bob | b@gmail.com | MacBook |

问题：

- 用户信息重复存储
- 修改邮箱需要改多行
- 容易产生数据不一致

---

### 规范化后（✅）

Users：

| user_id | name | email |
| --- | --- | --- |
| 1 | Alice | a@gmail.com |
| 2 | Bob | b@gmail.com |

Orders：

| order_id | user_id | product |
| --- | --- | --- |
| 1 | 1 | iPhone |
| 2 | 1 | AirPods |
| 3 | 2 | MacBook |

优点：用户信息单处存储、修改安全，结构更清晰。

---

## 三、三大常见范式（1NF / 2NF / 3NF）

### 1. 第一范式（1NF）

要求：每个字段必须是原子值（不可再拆分）。

示例：

- 错误：`name = "Alice, Bob"`
- 正确：`name = "Alice"`

---

### 2. 第二范式（2NF）

要求：所有非主键字段必须完全依赖整个主键（常见于复合主键），避免“部分依赖”。

---

### 3. 第三范式（3NF）

要求：不允许传递依赖。

例如：`order_id | user_id | user_email` 中，`user_email` 依赖 `user_id`，不应放在 Orders 中，应拆到 Users。

---

## 四、Normalization 的优缺点

### 优点（✅）

- 消除数据冗余，防止更新异常
- 一致性更高，结构更清晰

### 缺点（❌）

- 需要 JOIN，查询更复杂
- 读性能可能下降

---

## 五、Normalization vs Denormalization

### Normalization（规范化）

适用于：OLTP 系统、写操作较多、强一致性要求高、结构较复杂。

### Denormalization（反规范化）

适用于：读多写少、高并发、分布式数据库、需要减少 JOIN。

例如（NoSQL 文档存储）：

```json
{
  "user_id": 1,
  "name": "Alice",
  "email": "a@gmail.com",
  "orders": [
    { "product": "iPhone" },
    { "product": "AirPods" }
  ]
}
```

---

## 六、系统设计视角

Normalization 关注：数据正确性、更新安全性、结构清晰。

Denormalization 关注：查询性能、减少跨表 JOIN、分布式扩展效率。

---

## 七、设计时的判断标准

可以问自己：

> 该字段是否真正描述当前表的主键？

如果描述的是“另一个实体”，通常应拆为独立表。

---

## 八、一句话总结

> Normalization 为正确性与结构清晰；Denormalization 为性能与扩展性。
