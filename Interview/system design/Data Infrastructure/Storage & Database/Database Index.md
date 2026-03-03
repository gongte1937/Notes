# 数据如何存储到磁盘 & 有无 Index 时的读取流程总结

## 一、数据是如何存到磁盘的？

假设有一张表：

```sql
users(id, name, subbed)
```

当执行：

```sql
INSERT INTO users ...
```

数据库内部会发生以下事情：

### 1. 创建 Heap File（表文件）

每个表在磁盘上都会对应一个 **heap file**，这个文件专门用来存储真实数据。

---

### 2. 数据被切成 Block

数据库不会一条条散着存数据，而是：

- 把数据写入**固定大小的 Block（通常 8KB）**
- 每个 Block 里存放多条记录

示例结构：

```
Heap File
├── Block 1 (8KB)
│   ├── John
│   ├── Max
│   └── Alex
├── Block 2 (8KB)
│   ├── Chris
│   ├── Tom
│   └── Ed
└── ...
```

> Block 是数据库在磁盘上的最小读写单位。

---

## 二、没有 Index 时的查询流程

查询语句：

```sql
SELECT * FROM users WHERE name = 'Ed';
```

### 执行过程（Full Table Scan）

1. 读取 Block 1 到内存
2. 逐条比较 name
3. 如果没找到 → 读取 Block 2
4. 继续扫描
5. 直到找到或扫描完整张表

```
磁盘 → Block1 → Block2 → Block3 → ... → 找到
```

特点：

- 需要扫描大量 Block
- 数据量越大，越慢
- 这种方式叫：**Full Table Scan**

---

## 三、有 Index 时的存储结构

创建索引：

```sql
CREATE INDEX idx_name ON users(name);
```

此时磁盘上会新增一个 **Index File**，磁盘结构变成：

```
磁盘
├── Heap File（真实数据）
└── Index File（索引结构）
```

---

## 四、Index 里存什么？

Index 不存完整数据，它存：

```
数据值 → 数据所在位置
```

例如：

```
Ed   → (Block 2, Offset 3)
John → (Block 1, Offset 1)
Max  → (Block 1, Offset 2)
```

这些数据会被组织成 **B-Tree 结构**。

---

## 五、有 Index 时的查询流程

执行：

```sql
SELECT * FROM users WHERE name = 'Ed';
```

### 第一步：查 Index

1. 读取 Index File
2. 在 B-Tree 中快速定位
3. 找到 Ed
4. 获取对应位置：Block 2, Offset 3

### 第二步：直达数据 Block

5. 直接读取 Block 2
6. 找到第 3 条记录
7. 返回结果

```
磁盘 → Index → 得到 Block 地址 → 读取指定 Block
```

---

## 六、有无 Index 对比

| 情况      | 读取方式         | 读取 Block 数量 |
| --------- | ---------------- | --------------- |
| 无 Index  | 全表扫描         | 很多            |
| 有 Index  | 先查索引再直达   | 极少            |

---

## 七、Index Only Scan（进阶）

如果查询只涉及索引中的字段：

```sql
SELECT name FROM users WHERE name = 'Ed';
```

数据库可能只读取 Index File，而不访问 Heap File，这叫 **Index Only Scan**。

---

## 八、核心理解总结

| 概念        | 说明                         |
| ----------- | ---------------------------- |
| Block       | 物理存储单位                 |
| Heap File   | 表真实数据                   |
| Index File  | 加速查找的数据结构           |
| Index 内容  | 值 + 指向数据的指针          |
| 优化本质    | 减少磁盘 Block 读取次数      |
| 性能瓶颈    | 通常在磁盘 I/O               |

---

## 九、一句话理解

**没有 Index：** 一箱一箱翻仓库找东西

**有 Index：** 先查目录 → 直接走到指定货架拿

---

> Index 的本质是用空间换时间，通过减少磁盘 I/O 来提升查询性能。
