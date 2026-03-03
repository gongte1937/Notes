# Database Schema

The resume parser writes to tables in three Supabase schemas: `resume`, `student`, and `public`. For the pipeline step that performs these writes, see [Parsing Pipeline — Step 6](02-parsing-pipeline.md#step-6-database-storage-long-parse-only). For the TypeScript types that map to these tables, see [Data Models](03-data-models.md). For debugging queries, see [Troubleshooting](09-troubleshooting.md#debugging).

## Schema Overview

```
┌─────────────────────────────────────────────────────┐
│                   resume schema                     │
│                                                     │
│ resume_versions ◄───── resume_education             │
│       │                                             │
│       ├──────────────── resume_work_experience      │
│       │                                             │
│       └──────────────── resume_skills_queue         │
│                                                     │
│  parsing_jobs                                       │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                    public schema                    │
│                                                     │
│  freelance_work_history   (denormalized sync)       │
│  users                    (ownership + auth checks) │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   student schema                    │
│                                                     │
│  student_work_history                               │
│  student_education_history                          │
│  student_skills                                     │
└─────────────────────────────────────────────────────┘
```

## Resume Schema Tables

### `resume.resume_versions`

Links a user to their parsed resume data.

| Column | Type | Description |
|--------|------|-------------|
| `id` | int (PK) | Auto-increment primary key |
| `user_id` | uuid (FK) | References `users.id` |
| `is_active` | boolean | Whether this is the active resume version |
| `is_deleted` | boolean | Soft delete flag |
| `file_storage_path` | text | Path in Supabase Storage |
| `original_filename` | text | Original uploaded filename |
| `version_name` | text | User-friendly label |
| `file_hash` | text | SHA-256 hash for duplicate detection |
| `parsed_content` | jsonb | Full parsed data from AI (LongParseData) |
| `upload_date` | timestamp | When the file was uploaded |
| `created_at` | timestamp | Record creation time |
| `updated_at` | timestamp | Last update time (used for optimistic locking) |
| `deleted_at` | timestamp | Soft delete timestamp |

**Operations by parser:**
- UPDATE: Sets `parsed_content`, `is_active=true`, `updated_at`
- Uses optimistic locking on `updated_at` to prevent concurrent writes

**Operations by marketplace app:**
- INSERT: Creates record before calling parser (with file metadata)
- UPDATE: Sets `is_active=false` for all existing versions before creating new one

### `resume.resume_education`

Stores education records extracted from the resume.

| Column | Type | Description |
|--------|------|-------------|
| `id` | int (PK) | Auto-increment |
| `resume_version_id` | int (FK) | References `resume_versions.id` |
| `institution` | text | School/university name |
| `degree` | text | Mapped from `qualification` |
| `field_of_study` | text | Major/specialization |
| `start_date` | date | ISO date |
| `end_date` | date | ISO date (null if current) |
| `achievements` | text | Mapped from `grade` field |
| `created_at` | timestamp | Record creation time |

**Operations:** INSERT (non-blocking, continues on error)

### `resume.resume_work_experience`

Stores work history records extracted from the resume.

| Column | Type | Description |
|--------|------|-------------|
| `id` | int (PK) | Auto-increment, returned for skill linking |
| `resume_version_id` | int (FK) | References `resume_versions.id` |
| `position` | text | Job title (smart title cased) |
| `employer` | text | Company name |
| `responsibilities` | text | Array joined with `\n` |
| `achievements` | text | Array joined with `\n` |
| `start_date` | date | ISO date |
| `end_date` | date | ISO date (null if current) |
| `currently_working` | boolean | True if `date_end` is null/empty |
| `created_at` | timestamp | Record creation time |

**Operations:** INSERT with `.select("id")` to get IDs for skill linking

### `resume.resume_skills_queue`

Queues skills for downstream vectorization processing.

| Column | Type | Description |
|--------|------|-------------|
| `content` | text | Skill name (trimmed) |
| `freelancer_id` | uuid | User ID |
| `experience` | int | Years of experience (preserves 0 via `??`) |
| `qualificationLevel` | text | Proficiency level |
| `category` | text | Skill category |
| `skill_type` | text | Always null (not provided by parser) |
| `workHistoryId` | int | Always null (linking deferred) |
| `qualificationId` | int | Always null (linking deferred) |

**Operations:** INSERT (non-blocking, filters out null `content`)

### `resume.parsing_jobs`

Tracks async parsing job status and results.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Job ID (generated by marketplace app) |
| `user_id` | uuid | User who initiated the parse |
| `resume_url` | text | URL of the resume file |
| `status` | text | Job status (see below) |
| `error` | text | Error message if failed |
| `parsed_data` | jsonb | Full parsed response from Azure Function |
| `updated_at` | timestamp | Last status change |

**Job Status Values:**

| Status | Meaning | Set By |
|--------|---------|--------|
| `pending` | Created, not yet processing | Marketplace app |
| `running` | Processing in progress | Marketplace app |
| `completed` | Successfully parsed and stored | Marketplace app (background) |
| `completed_with_errors` | Parsed but student storage failed | Marketplace app (background) |
| `failed` | Processing failed | Marketplace app (background) |
| `cancelled` | Superseded by newer job | Marketplace app |
| `read` | Results consumed by client | Marketplace app (status endpoint) |

**Operations by parser:** UPDATE `status="completed"` (if jobId provided)

**Operations by marketplace:** INSERT, UPDATE (status transitions), SELECT (polling)

## Public Schema Tables

### `public.freelance_work_history`

Denormalized work history for freelancer profiles. The parser syncs this from `resume_work_experience` after successful insertion.

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | uuid | User ID |
| `resume_version_id` | int | References `resume_versions.id` |
| `position` | text | Job title (smart title cased) |
| `employer` | text | Company name |
| `responsibilities` | text | Array joined with `\n` |
| `achievements` | text | Array joined with `\n` |
| `startDateMonth` | text | Start month (1-12) |
| `startDateYear` | text | Start year |
| `endDateMonth` | text | End month (null if current) |
| `endDateYear` | text | End year (null if current) |
| `currentlyWorking` | boolean | True if no end date |
| `industryHashTag` | text | Always null (set later) |
| `totalExperience` | text | Duration as "Xy Ym" (e.g., "3y 6m") |
| `created_at` | timestamp | Record creation time |

**Operations:** INSERT (non-blocking, continues on error)

### `public.users`

Provides read-only access for authorization checks.

| Column | Type | Used For |
|--------|------|----------|
| `id` | uuid | Ownership validation |
| `auth_user_id` | uuid | Supabase Auth user mapping |
| `userType` | text | Student authorization (`studentUser` check) |

**Operations:** SELECT (ownership check, student authorization)

## Student Schema Tables

The parser writes to these tables only when `isStudent: true` AND the user has `userType: 'studentUser'`. See [Workflows — Student Parsing](06-workflows.md#student-parsing-flow) for the end-to-end student flow and [Security — Student Authorization](07-security-and-config.md#student-authorization) for the authorization check.

### `student.student_work_history`

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | uuid | User ID |
| `resume_version_id` | int | References `resume_versions.id` |
| `position` | text | Job title (smart title cased) |
| `employer` | text | Company name |
| `startDateMonth` | text | Start month |
| `startDateYear` | text | Start year |
| `endDateMonth` | text | End month |
| `endDateYear` | text | End year |
| `currentlyWorking` | boolean | True if no end date |
| `totalExperience` | text | Duration as "Xy Ym" |
| `industryHashTag` | text | Always null |

### `student.student_education_history`

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | uuid | User ID |
| `resume_version_id` | int | References `resume_versions.id` |
| `qualification` | text | Degree/qualification |
| `field_of_study` | text | Major/specialization |
| `institution` | text | School/university |
| `startDateMonth` | text | Start month |
| `startDateYear` | text | Start year |
| `endDateMonth` | text | End month |
| `endDateYear` | text | End year |
| `currently_studying` | boolean | True if no end date |
| `certificate_url` | text | Always null |
| `totalExperience` | text | Always null |

### `student.student_skills`

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | uuid | User ID |
| `skillname` | text | Skill name (trimmed) |
| `experience` | int | Years of experience |
| `qualificationLevel` | text | Proficiency level |
| `category` | text | Skill category |
| `workHistoryId` | int | Always null (linking deferred) |
| `educationId` | int | Always null (linking deferred) |
| `norm_skillname` | text | Always null |

## Write Order and Error Strategy

```
1. resume_versions       ──► UPDATE (BLOCKING - fails entire operation)
2. parsing_jobs          ──► UPDATE (non-blocking)
3. resume_education      ──► INSERT (non-blocking)
4. resume_work_experience ──► INSERT (non-blocking, returns IDs)
5. freelance_work_history ──► INSERT (non-blocking)
6. resume_skills_queue   ──► INSERT (non-blocking)
7. student_work_history  ──► INSERT (non-blocking, if isStudent)
8. student_education     ──► INSERT (non-blocking, if isStudent)
9. student_skills        ──► INSERT (non-blocking, if isStudent)
```

"Non-blocking" means the parser logs errors but continues the overall operation. The response includes `storageSuccess: true` as long as step 1 succeeds.

## Optimistic Locking Detail

The parser prevents concurrent writes to a resume version:

1. **Read** current `updated_at` from `resume_versions`
2. **Update** with `WHERE updated_at = <read value>`
3. If `count === 0`: re-fetch to determine cause
   - `is_deleted === true` → "Resume was deleted"
   - Timestamp mismatch → "Resume was modified by another request. Please retry."

This prevents data loss when two parsing jobs finish simultaneously for the same resume.
