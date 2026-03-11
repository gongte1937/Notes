# Parsing Pipeline

The resume parser processes files through a sequential pipeline. Each step transforms data and passes it to the next. For request/response type definitions, see [Data Models](03-data-models.md). For database table details, see [Database Schema](04-database-schema.md).

## Pipeline Overview

```
Request ──► Validation ──► File Fetch ──► Text Extraction ──► AI Processing ──► Response Cleanup ──► Database Storage
              │                │               │                   │                  │                    │
           API key          S3 Func       pdf-parse/          OpenAI GPT         JSON/CSV              Supabase
           + fields         + decrypt     mammoth/OCR         (parallel)         normalization          (multi-table)
```

## Step 1: Request Validation

**Source:** `src/functions/validators/requestValidator.ts` | See [Security & Configuration](07-security-and-config.md) for Key Vault details

### API Key Validation
- Reads `resume-parser-api-key` from Azure Key Vault (cached in-memory)
- Compares it against the `X-API-Key` request header
- Returns `401` if the key is missing or invalid, `503` if Key Vault is unavailable

### Field Validation

**Short parse** requires: `url`, `fileType`, `env`, `encrypted`

**Long parse** additionally requires: `userId` (validated as UUID v1-v5)

Valid file types: `pdf`, `docx`, `txt`, `md`

Valid environments: `development`, `test`, `production`

## Step 2: File Fetch

**Source:** `src/functions/processors/fileProcessor.ts` → `readS3File()`

The parser fetches the resume file from Supabase Storage via the S3 Functions service.

```
Parser ──POST──► af-dv-s3-functions.azurewebsites.net/api/read-file-s3
                  │
                  │  Request: { url, fileType, env, encrypted }
                  │  Headers: { x-functions-key: <from Key Vault> }
                  │
                  ◄── Response: { content: <base64>, fileName, mimeType }
```

- File type mapping: `pdf` → `PDF`, `docx` → `DOCX`, `txt` → `TXT`, `md` → `MD`
- The `encrypted` flag tells S3 Functions to decrypt via CipherStash before returning
- The parser decodes response content from base64 into a `Buffer`

## Step 3: Text Extraction

**Source:** `src/functions/processors/fileProcessor.ts`

The extraction method depends on file type:

### TXT / MD Files
- Direct `Buffer.toString('utf-8')` conversion
- Skips OCR entirely (faster processing)

### PDF Files
- **Short parse:** Uses `pdf-parse` library to extract text from buffer
- **Long parse:** Uses Azure Document Intelligence OCR (`prebuilt-read` model)

### DOCX Files
- **Short parse:** Uses `mammoth` library (detected as ZIP by mimetics)
  - Writes to temp file, extracts text, cleans up temp file
- **Long parse:** Uses Azure Document Intelligence OCR

### Azure Document Intelligence OCR

**Source:** `fileProcessor.ts` → `azureDocumentIntelligenceOcr()`

The **long parser** uses this for PDF and DOCX files:

```typescript
const client = new DocumentAnalysisClient(endpoint, new AzureKeyCredential(apiKey));
const poller = await client.beginAnalyzeDocument('prebuilt-read', fileData.content);
const result = await poller.pollUntilDone();
```

- Async polling pattern
- Credentials from Key Vault: `document-intelligence-endpoint`, `document-intelligence-key`

### Text Normalization

After extraction, `prepareNewlines()` replaces `\n` with ` \n ` so the LLM parses line-delimited content more accurately.

## Step 4: AI Processing

**Source:** `src/functions/processors/aiProcessor.ts` → `callOpenAI()`

### Short Parse (Single Call)

One OpenAI call with `SHORT_PARSE` prompt:
- Extracts: firstName, lastName, address, city, zipCode, stateProvince, country, dateOfBirth, mobileNumber, gender
- Normalizes phone numbers to `+61123456789` or `0123456789` format
- Response format: JSON object

### Long Parse (Two Parallel Calls)

Uses `Promise.allSettled()` for parallel execution:

```
                          ┌──────────────────────┐
                          │  callOpenAI("long")  │──► JSON with work_history,
         cleanedText ─────┤                      │    education, certifications,
                          │ callOpenAI("skills") │──► CSV with skill_name,
                          └──────────────────────┘    category, proficiency, years
```

**Main parse** (`LONG_PARSE` prompt):
- Response format: `json_object` (OpenAI enforces this)
- Extracts: work_history, education_history, certifications, projects
- Dates in ISO 8601 format (YYYY-MM-DD)

**Skills parse** (`LONG_SKILLS_PARSE` prompt):
- Response format: CSV (no JSON constraint)
- Extracts up to 35 skills with normalization rules:
  - Lowercase everything
  - Remove version numbers (`python 3.7` → `python`)
  - Expand acronyms (`ml` → `machine learning`)
  - Map synonyms (`reactjs` → `react`, `postgres` → `postgresql`)

### OpenAI Configuration

| Setting | Value |
|---------|-------|
| Deployment | `gpt-5-mini` |
| API Version | `2025-01-01-preview` |
| Max Retries | 3 |
| Backoff Base | 5,000ms |
| Backoff Multiplier | 1.5x |
| Retry Delays | 5s → 7.5s → 11.25s |

### Error Handling

- If main parse fails: throws error (entire request fails)
- If skills parse fails: continues with empty skills array
- Retries on any error (network, rate limit, server error)

## Step 5: Response Cleanup

**Source:** `src/functions/processors/aiProcessor.ts`

### JSON Cleanup (`cleanupLlmResponse`)

1. Strip text before first `{` and after last `}`
2. Remove trailing commas before `}` or `]`
3. Parse as JSON
4. Recursively replace empty arrays `[]` with `null`

### CSV Cleanup (`parseCsvToJson`)

1. Check whether the first line matches the expected header: `skill_name,skill_category,proficiency_level,years_of_experience`
2. Normalize a close-but-inexact header (case-insensitive matching)
3. Prepend the expected header if none exists
4. Parse rows into objects
5. Convert `years_of_experience` to integer (default 0 if invalid)
6. Skip malformed rows (wrong column count)
7. Return empty array on total failure (non-blocking)

### Data Merge

The processor merges the skills array into the main parsed data object:
```typescript
cleanedData.skills = skills.length > 0 ? skills : null;
```

## Step 6: Database Storage (Long Parse Only)

**Source:** `src/functions/processors/databaseProcessor.ts` → `storeToSupabase()`

The short parser returns data directly without database storage. The long parser stores results across multiple tables. See [Database Schema](04-database-schema.md) for full table definitions and write order.

### Freelancer Tables (default)

The processor executes these in sequence:

```
1. resume_versions      ──► UPDATE parsed_content, is_active=true (optimistic lock)
2. resume_education     ──► INSERT education records
3. resume_work_experience ──► INSERT work records (returns IDs for skill linking)
4. freelance_work_history ──► INSERT denormalized work records (public schema)
5. resume_skills_queue  ──► INSERT skills for vectorization pipeline
6. parsing_jobs         ──► UPDATE status="completed" (if jobId provided)
```

### Student Tables (when `isStudent: true`)

After freelancer storage, if `isStudent` is true:

```
1. Validate student authorization (users.userType === 'studentUser')
2. student_work_history        ──► INSERT work records (student schema)
3. student_education_history   ──► INSERT education records (student schema)
4. student_skills              ──► INSERT skills (student schema)
```

### Optimistic Locking

The `resume_versions` update uses optimistic locking:

```typescript
// 1. Read current updated_at timestamp
const { data: ownershipCheck } = await supabase
  .from("resume_versions")
  .select("updated_at")
  .eq("id", resumeVersionId)
  .eq("user_id", userId)
  .eq("is_deleted", false)
  .single();

// 2. Update with timestamp check
const { count } = await supabase
  .from("resume_versions")
  .update({ parsed_content, is_active: true, updated_at: nowIso })
  .eq("id", resumeVersionId)
  .eq("updated_at", currentUpdatedAt);  // ← optimistic lock

// 3. If count === 0, concurrent modification detected
```

### Non-Blocking Subsidiary Inserts

Education, work experience, skills, and freelance_work_history inserts continue on error. Only a `resume_versions` update failure blocks the pipeline.

### Data Transformations

| Field | Transformation |
|-------|---------------|
| Job titles | Converted to smart title case (preserves acronyms like AWS, CEO) |
| Dates | Parsed as ISO 8601 via `parseISODate()` |
| Responsibilities | Joined from array with `\n` |
| Achievements | Joined from array with `\n` |
| Duration | Calculated as `Xy Ym` format (e.g., `3y 6m`) |
| Currently working | Set to `true` if `date_end` is null or empty |
| Skill names | Filtered for non-null, trimmed |
| Years of experience | Preserved via nullish coalescing (`??`) to keep `0` values |
