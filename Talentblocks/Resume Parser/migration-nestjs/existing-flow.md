# Resume Parser 流程

> **当前架构**：Resume Parser 服务基于 **Azure Functions**（Serverless）实现。
> **后续计划**：将整个 Resume Parser 迁移至 **NestJS**，统一后端服务架构，详见本文件夹其他迁移文档。

## 一、前端上传

用户在 Talentblocks 选择简历文件，前端先计算文件 hash 做去重，然后上传到 **Supabase Storage**，获取文件的 `url/path`。

## 二、前端调用 Next.js API Routes

前端调用 Marketplace 的 Next.js API routes，分两类：

| 路由 | 模式 | 说明 |
| --- | --- | --- |
| `POST /api/parse/short` | 同步 | 快速提取联系人等个人信息，直接返回结果 |
| `POST /api/parse/long/start` | 异步 | 创建 `parsing_jobs`、创建/切换 `resume_versions`，后台触发完整解析，立刻返回 `jobId` |

## 三、Next.js 调用 Azure Functions Resume Parser

### Short 解析

```
读文件文本 → 调 Azure OpenAI 做短解析 → 清洗 → 直接返回（不落库）
```

### Long 解析

```
POST /api/read-file-s3（S3 Functions）
  → 从 Supabase Storage 读取文件（如需则解密）
  → pdf/docx 用 Azure Document Intelligence OCR 转文本
  → 并行调用 Azure OpenAI
      ├── 主内容 JSON
      └── 技能 CSV
  → 清洗合并
  → 写入 Supabase 多张表（含乐观锁，必要时写 student schema）
  → 更新 job 状态
```

## 四、前端轮询结果

前端持续轮询 `GET /api/parse/long/status?jobId=...`，直到状态完成；完成后读取 Supabase 的结构化数据并展示在简历页面。
