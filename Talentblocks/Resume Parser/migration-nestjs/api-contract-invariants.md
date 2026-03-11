# Resume Parser API Contract Invariants

> 目的：作为 NestJS migration 期间的兼容性基线。  
> 结论：以下接口与数据结构在迁移后应保持不变（除非做版本化变更）。

## Scope

- 前端调用接口：`POST /api/parse/short`、`POST /api/parse/long/start`
- 最终返回前端的数据形态（含 long status 阶段）
- 解析过程中写入 Supabase 的关键数据类型

## 1) `POST /api/parse/short`

### Frontend -> Marketplace API Request

```ts
interface ParseShortRequest {
  fileUrl: string;
  fileType: string; // e.g. "pdf"
  userId: string; // uuid
  isStudent: boolean;
}
```

### Marketplace API -> Frontend Response

```ts
interface ParseShortResponse {
  personal: {
    firstName?: string;
    lastName?: string;
    email?: string;
    mobileNumber?: string;
    address?: string;
    apartmentSuite?: string;
    city?: string;
    stateProvince?: string;
    country?: string;
    zipCode?: string | number; // 文档中示例与模型存在 string/number 差异，兼容处理
    dateOfBirth?: string;
    gender?: string;
  };
}
```

### Internal Forwarding (Marketplace -> Azure Short Parser)

```ts
interface AzureShortParseRequest {
  url: string;
  fileType: "txt" | "md" | "pdf" | "docx";
  env: string;
  encrypted: boolean;
  isStudent: boolean;
}
```

## 2) `POST /api/parse/long/start`

### Frontend -> Marketplace API Request

```ts
interface ParseLongStartRequest {
  fileUrl: string;
  fileType: string;
  userId: string;
  fileName: string;
  fileHash: string; // sha256
  isStudent: boolean;
}
```

### Marketplace API -> Frontend Immediate Response

```ts
interface ParseLongStartResponse {
  jobId: string; // uuid
}
```

`/api/parse/long/start` 不直接返回完整解析结果。完整结果通过后续轮询 `GET /api/parse/long/status?jobId=...` 获取。

### Long Status Response (Final data consumed by frontend)

```ts
type ParsingStatus =
  | "pending"
  | "running"
  | "completed"
  | "completed_with_errors"
  | "failed"
  | "read";

interface LongParseData {
  user?: {
    first_name?: string;
    last_name?: string;
    middle_name?: string;
    preferred_name?: string;
    email?: string;
    country?: string;
    city?: string;
    phone_number?: string;
  };
  skills?: Array<{
    skill_name?: string;
    skill_category?: string;
    proficiency_level?: string;
    years_of_experience?: number;
  }>;
  work_history?: Array<{
    job_title?: string;
    company?: string;
    responsibilities?: string[];
    achievements?: string[];
    date_start?: string;
    date_end?: string;
  }>;
  education_history?: Array<{
    institution?: string;
    qualification?: string;
    field_of_study?: string;
    date_start?: string;
    date_end?: string;
    grade?: string;
  }>;
  certifications?: Array<{
    certification_name?: string;
    issuing_organization?: string;
    date_obtained?: string;
    expiry_date?: string;
    verification_link?: string;
  }>;
  projects?: Array<{
    project_client?: string;
    project_name?: string;
    project_description?: string;
    technology_used?: string[];
    project_start?: string;
    project_end?: string;
    project_role?: string;
    project_responsibilities?: string[];
  }>;
}

interface LongParseResult {
  success: boolean;
  data?: LongParseData;
  error?: string;
  processingTime?: number;
  processedAt: Date;
  storageSuccess?: boolean;
  storageError?: string;
  jobUpdateSuccess?: boolean;
  jobUpdateError?: string;
  studentStorageSuccess?: boolean;
  studentStorageError?: string;
}

interface ParseLongStatusResponse {
  status: ParsingStatus;
  error?: string | null;
  user_id: string;
  data?: LongParseResult;
}
```

## 3) Supabase Write Types (In-Between Data)

### 3.1 Core JSONB payloads

| Table | Column | Type | Value Shape |
| --- | --- | --- | --- |
| `resume.parsing_jobs` | `parsed_data` | `jsonb` | `LongParseResult` |
| `resume.resume_versions` | `parsed_content` | `jsonb` | `LongParseData` |

### 3.2 Job and version metadata

### `resume.parsing_jobs`

| Column | Type |
| --- | --- |
| `id` | `uuid` |
| `user_id` | `uuid` |
| `resume_url` | `text` |
| `status` | `text` |
| `error` | `text` |
| `parsed_data` | `jsonb` |
| `updated_at` | `timestamp` |

### `resume.resume_versions`

| Column | Type |
| --- | --- |
| `id` | `int` |
| `user_id` | `uuid` |
| `is_active` | `boolean` |
| `is_deleted` | `boolean` |
| `file_storage_path` | `text` |
| `original_filename` | `text` |
| `version_name` | `text` |
| `file_hash` | `text` |
| `parsed_content` | `jsonb` |
| `upload_date` | `timestamp` |
| `created_at` | `timestamp` |
| `updated_at` | `timestamp` |
| `deleted_at` | `timestamp` |

### 3.3 Structured parsed tables

- `resume.resume_education`：`institution/degree/field_of_study` 等 `text` 字段，`start_date/end_date` 为 `date`
- `resume.resume_work_experience`：`position/employer` 等 `text`，`start_date/end_date` 为 `date`，`currently_working` 为 `boolean`
- `resume.resume_skills_queue`：技能文本与标签为 `text`，`experience` 为 `int`
- `public.freelance_work_history`：对 work history 的反规范化结果，混合 `text/boolean/timestamp`
- 学生分支（`isStudent=true` 且授权通过）写入 `student.student_work_history`
- 学生分支（`isStudent=true` 且授权通过）写入 `student.student_education_history`
- 学生分支（`isStudent=true` 且授权通过）写入 `student.student_skills`

## 4) Migration Non-Change Checklist

- 路由路径保持不变：`/api/parse/short`、`/api/parse/long/start`、`/api/parse/long/status`
- `POST /api/parse/short` 的请求字段与 `personal` 返回结构保持兼容
- `POST /api/parse/long/start` 继续“立即返回 `{ jobId }`”
- 完整长解析结果继续通过 `status` 接口返回
- `parsing_jobs.status` 枚举值保持兼容：`pending/running/completed/completed_with_errors/failed/cancelled/read`
- `resume.parsing_jobs.parsed_data` 与 `resume.resume_versions.parsed_content` 继续分别承载 `LongParseResult` 与 `LongParseData`
- 现有列名与基础类型（uuid/int/text/boolean/date/timestamp/jsonb）不变

## References

- `Talentblocks/Resume Parser/Resume Parser Documentation/03-data-models.md`
- `Talentblocks/Resume Parser/Resume Parser Documentation/04-database-schema.md`
- `Talentblocks/Resume Parser/Resume Parser Documentation/05-marketplace-integration.md`
- `Talentblocks/Resume Parser/Resume Parser Documentation/06-workflows.md`
