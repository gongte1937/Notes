# Troubleshooting

For the pipeline steps referenced below, see [Parsing Pipeline](02-parsing-pipeline.md). For database table definitions, see [Database Schema](04-database-schema.md). For monitoring and Application Insights setup, see [Deployment & Testing](08-deployment-and-testing.md#monitoring).

## HTTP Status Codes

### Parser Service (Azure Functions)

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| 200 | Success | Full completion |
| 207 | Partial success | Parsing succeeded but database storage or job update failed |
| 400 | Bad request | Missing fields, invalid file type, empty document, no text extracted |
| 401 | Unauthorized | Missing or invalid `X-API-Key` header |
| 500 | Server error | Processing failure (OCR, AI, or unhandled exception) |
| 503 | Service unavailable | Key Vault or authentication service down |

### Marketplace API Routes

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| 200 | Success | Parse completed |
| 400 | Bad request | Missing `jobId` parameter |
| 401 | Unauthorized | Missing Bearer token, invalid session, or userId mismatch |
| 404 | Not found | Job ID doesn't exist in `parsing_jobs` |
| 500 | Server error | Azure Function unreachable, Supabase error |

## Common Issues

### "No text content found in document" (400)

**Cause:** The uploaded file contains no extractable text. Common sources:
- Image-only PDFs (scanned without OCR)
- Corrupt or empty files
- Protected/encrypted PDFs that block text extraction

**Resolution:**
- Ask user to upload a text-based PDF or DOCX
- Check that the file isn't password-protected

### "Resume version not found or access denied"

**Cause:** No record matches the given `resumeVersionId` with that `userId` and `is_deleted=false`.

**Resolution:**
- Verify the marketplace app created the `resume_versions` record before calling the parser
- Confirm the record is still active (not soft-deleted)

### "Resume was modified by another request. Please retry."

**Cause:** Another process updated the `resume_versions` record between the read and write, triggering the optimistic lock.

**Resolution:**
- The frontend should retry the operation
- Check for duplicate parsing jobs (concurrent job cancellation should prevent these)

### "All OpenAI retry attempts failed"

**Cause:** Azure OpenAI returned errors on all three retry attempts.

**Resolution:**
- Check Azure OpenAI service health
- Verify the `gpt-5-mini` deployment is active
- Check rate limits on the Azure OpenAI resource
- Review Application Insights for specific error codes

### 207 Multi-Status Response

**Cause:** Parsing succeeded but at least one database operation failed.

**What to check:**
- `storageSuccess: false` + `storageError` → Resume version update failed
- `jobUpdateSuccess: false` + `jobUpdateError` → parsing_jobs update failed
- `studentStorageSuccess: false` + `studentStorageError` → Student table writes failed

**Resolution:**
- The response body still contains the parsed data
- Check Supabase connectivity and RLS policies
- For student errors: verify the user has `userType: 'studentUser'`

### Parsing Job Stuck in "running" Status

**Cause:** The background `processResumeAsync()` function likely timed out or crashed. See [Workflows — Job Recovery](06-workflows.md#job-recovery-page-refresh) for the frontend recovery mechanism.

**Resolution:**
1. Check Azure Function logs in Application Insights
2. Verify the 5-minute timeout hasn't been exceeded
3. The marketplace app cancels stale jobs when initiating a new parse
4. The frontend's localStorage recovery ignores jobs older than 24 hours

### Short Parse Returns Empty Fields

**Cause:** The AI found no matching information in the resume text.

**Resolution:**
- The short parser extracts only explicitly stated information
- Gender extraction requires an explicit statement (the parser does not infer)
- Phone numbers normalize to `+61...` or `0...` format
- ZipCode defaults to `0` when absent

### Student Tables Not Populated

**Cause:** Student table writes require both `isStudent: true` in the request AND `userType: 'studentUser'` in the `users` table.

**Resolution:**
1. Verify the request passes `isStudent: true`
2. Check `users.userType` for the user
3. If `userType` differs from `studentUser`, the parser intentionally skips student tables

## Debugging

### Application Insights Queries

**Recent parser invocations:**
```kusto
requests
| where name contains "parse"
| order by timestamp desc
| take 50
```

**Failed requests:**
```kusto
requests
| where success == false
| where name contains "parse"
| order by timestamp desc
```

**OpenAI latency:**
```kusto
traces
| where message contains "parser took"
| project timestamp, message
| order by timestamp desc
```

**Error traces:**
```kusto
exceptions
| where timestamp > ago(24h)
| order by timestamp desc
```

### Checking Job Status Directly

Query the Supabase `parsing_jobs` table:

```sql
SELECT id, user_id, status, error, updated_at
FROM resume.parsing_jobs
WHERE user_id = 'user-uuid-here'
ORDER BY updated_at DESC
LIMIT 5;
```

### Verifying Resume Version Data

```sql
SELECT id, user_id, is_active, is_deleted, version_name,
       parsed_content IS NOT NULL as has_parsed_data,
       updated_at
FROM resume.resume_versions
WHERE user_id = 'user-uuid-here'
ORDER BY created_at DESC;
```

### Checking Skill Queue

```sql
SELECT content, experience, "qualificationLevel", category
FROM resume.resume_skills_queue
WHERE freelancer_id = 'user-uuid-here'
ORDER BY id DESC
LIMIT 20;
```

## Performance Characteristics

| Operation | Typical Duration |
|-----------|-----------------|
| File fetch (S3 Functions) | 1-3s |
| Text extraction (pdf-parse/mammoth) | <1s |
| OCR (Azure Document Intelligence) | 2-5s |
| OpenAI short parse | 2-4s |
| OpenAI long parse | 4-8s |
| OpenAI skills parse | 3-6s |
| Database storage (all tables) | 1-3s |
| **Total short parse** | **3-8s** |
| **Total long parse** | **8-20s** |

The long parser runs two OpenAI calls in parallel, so AI processing time equals the slower call, not the sum.

## Log Patterns

The parser uses consistent log prefixes for easy searching:

| Pattern | Component |
|---------|-----------|
| `Short parser request received` | httpTriggers |
| `Long parser request received` | httpTriggers |
| `Reading S3 file:` | fileProcessor |
| `Processing PDF file` / `Processing DOCX file` | fileProcessor |
| `Skipping OCR for text file` | fileProcessor |
| `Calling OpenAI for {type} parse (attempt {n}/{max})` | aiProcessor |
| `{type} parser took {n} seconds` | aiProcessor |
| `Parsing CSV to JSON` | aiProcessor |
| `Starting Supabase storage process` | databaseProcessor |
| `Verified ownership:` | databaseProcessor |
| `Concurrent modification detected` | databaseProcessor |
| `Starting freelance_work_history sync` | databaseProcessor |
| `Starting student_work_history sync` | databaseProcessor |
