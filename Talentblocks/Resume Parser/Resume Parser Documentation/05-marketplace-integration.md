# Marketplace Integration

How the Next.js marketplace app triggers, monitors, and consumes parsed resume data. For end-to-end user workflows (onboarding, updates, version management), see [Workflows](06-workflows.md). For the Azure Functions pipeline these routes invoke, see [Parsing Pipeline](02-parsing-pipeline.md).

## API Routes

Four Next.js API routes handle the resume parser integration:

```
src/app/api/parse/
├── short/route.ts              # POST - Synchronous contact info extraction
└── long/
    ├── start/route.ts          # POST - Start async parsing job
    ├── status/route.ts         # GET  - Poll job status
    └── run-sync/route.ts       # POST - Synchronous full parse (testing)
```

### POST `/api/parse/short`

**Source:** `src/app/api/parse/short/route.ts`

Calls the Azure Short Parser synchronously and returns extracted contact information.

**Request body:**
```json
{ "fileUrl": "...", "fileType": "pdf", "userId": "uuid", "isStudent": false }
```

**Flow:**
1. Validate Bearer token via Supabase Auth
2. Verify `userId` belongs to authenticated user (`users.auth_user_id` check)
3. Call Azure Short Parser (`AZURE_SHORT_PARSER_URL`)
4. Return normalized personal data

**Response:**
```json
{
  "personal": {
    "firstName": "John", "lastName": "Doe",
    "email": "", "mobileNumber": "+61400123456",
    "address": "123 Main St", "city": "Sydney",
    "stateProvince": "NSW", "country": "Australia",
    "zipCode": "2000", "dateOfBirth": "", "gender": ""
  }
}
```

### POST `/api/parse/long/start`

**Source:** `src/app/api/parse/long/start/route.ts`

Starts an asynchronous parsing job and returns a `jobId` for polling.

**Request body:**
```json
{
  "fileUrl": "...", "fileType": "pdf", "userId": "uuid",
  "fileName": "resume.pdf", "fileHash": "sha256...", "isStudent": false
}
```

**Flow:**
1. Validate Bearer token via Supabase Auth
2. Verify `userId` belongs to authenticated user
3. Cancel any existing `pending`/`running` jobs for the user
4. Create new `parsing_jobs` record with status `running`
5. Deactivate all existing `resume_versions` for the user
6. Create new `resume_versions` record (active, with file metadata)
7. Fire `processResumeAsync()` in background (no await)
8. Return `{ jobId }` immediately

**Background processing** (`processResumeAsync`):
- Calls Azure Long Parser with 5-minute timeout
- On success: updates `parsing_jobs` with status `completed` and `parsed_data`
- On student storage failure: sets status `completed_with_errors`
- On error: sets status `failed` with error message

### GET `/api/parse/long/status`

**Source:** `src/app/api/parse/long/status/route.ts`

Returns parsing job status. The frontend polls this endpoint repeatedly.

**Query params:** `?jobId=uuid`

**Flow:**
1. Query `parsing_jobs` for the job
2. If completed: include `parsed_data` in response and mark job as `read`
3. Return status, error, user_id, and data (if completed)

**Response:**
```json
{
  "status": "completed",
  "error": null,
  "user_id": "uuid",
  "data": { /* full LongParseResult */ }
}
```

**TTL cleanup:** The route marks jobs as `read` rather than deleting them, avoiding race conditions. A separate TTL-based cleanup job handles deletion.

### POST `/api/parse/long/run-sync`

**Source:** `src/app/api/parse/long/run-sync/route.ts`

Runs a synchronous full parse for testing or fallback scenarios with a 60-second timeout.

**Request body:**
```json
{ "fileUrl": "...", "fileType": "pdf", "userId": "uuid" }
```

**Response:**
```json
{ "data": { /* full LongParseResult */ } }
```

## Authentication Pattern

All parse routes follow the same auth pattern (see [Security & Configuration](07-security-and-config.md) for the full authentication chain):

```typescript
// 1. Extract Bearer token
const authHeader = request.headers.get('authorization');
const token = authHeader.replace('Bearer ', '');

// 2. Validate user session via Supabase Auth
const userSupabase = createClient(url, anonKey, {
  global: { headers: { Authorization: `Bearer ${token}` } }
});
const { data: { user } } = await userSupabase.auth.getUser();

// 3. Verify userId ownership
const { data: usersRow } = await supabase
  .from('users')
  .select('id')
  .eq('id', userId)
  .eq('auth_user_id', user.id)
  .single();
```

## Frontend Components

### Freelancer Resume Page

**Source:** `src/app/freelancer/resume/page.tsx`

Primary page for resume upload and parsing. Key functions include:

| Function | Purpose |
|----------|---------|
| `documentUploader()` | Upload resume to Supabase Storage (`contractorResume/` bucket) |
| `handleSubmitOnboarding()` | Run dual parsing: short (personal) + long (comprehensive) |
| `handleSubmitUpdate()` | Run long parsing only for profile updates |
| `startParsing()` | Initiate both short and long parsing simultaneously |
| `scheduleStatusPolling()` | Set up polling with initial delay and interval checks |
| `checkParsingStatus()` | Fetch job status from `/api/parse/long/status` |
| `savePersonalDetails()` | Save short parser results to `freelancer_personal_details` via encryption API |
| `loadExistingResumes()` | Fetch all `resume_versions` for the user |
| `confirmUpdateActive()` | Toggle resume active status and sync work history |
| `confirmDelete()` | Soft-delete resume (set `is_deleted=true`) |

### Student Resume Page

**Source:** `src/app/student/resume/page.tsx`

Follows the same flow as the freelancer page but stores data in `student` schema tables.

### Resume Manager

**Source:** `src/app/freelancer/manage-resumes/page.tsx`

Manages multiple resume versions: listing, toggling active status, and deleting.

## File Upload Flow

```
User selects file
      │
      ▼
Compute SHA-256 hash (client-side)
      │
      ▼
Check for duplicate (same hash = already uploaded)
      │
      ▼
Upload to Supabase Storage
  Bucket: ${NEXT_PUBLIC_BUCKET_NAME}
  Path: contractorResume/{timestamp}-{sanitizedFilename}
      │
      ▼
Get public URL
  ${NEXT_PUBLIC_BUCKET_URL}{storagePath}
      │
      ▼
Pass URL to parsing API
```

## Progress Tracking

The frontend displays a simulated progress bar during parsing:

| Time | Progress | Phase |
|------|----------|-------|
| 0s | 5% | Parsing started |
| 0-15s | 5% → 60% | Processing file and AI extraction |
| 15-40s | 60% → 95% | Database storage |
| Completion | 100% | Results loaded |
| +2s | Reset | Progress bar hidden |

## Redux State Management

**Source:** `src/redux/reducer/userReducer.ts`

Parser-related Redux actions:

| Action | Purpose |
|--------|---------|
| `setParserInitalData` | Store initial parsing response |
| `setParserFinalData` | Store final parsing results |
| `setLoaderForParser` | Toggle parser loading state |
| `setUserPersonalDetail` | Store short parse personal info |
| `setUserProfessionalDetail` | Store professional summary and years of experience |
| `setUserSkills` | Store skills array |
| `setWorkHistory` | Store work experience array |
| `setProgressBarPercentage` | Update profile completion progress |

## Custom Hooks

### `useParsingStatus`

**Source:** `src/app/freelancer/_hooks/useParsingStatus.ts`

React hook that tracks parsing job status with automatic polling.

```typescript
const { status, isPolling, jobId, recheckStatus } = useParsingStatus();
```

- Retrieves `jobId` from URL params or localStorage
- Polls `/api/parse/long/status` at a configurable interval (default 4s)
- Fetches professional data automatically on completion
- Cleans up and handles error states

## Parsing Helpers

### `fetchAndLoadProfessionalData()`

**Source:** `src/helper/parsing/parsingHelpers.ts`

Loads parsed professional data into Redux after parsing completes.
1. Fetch active resume version ID
2. Query work_history, skills, and professional_details in parallel
3. Filter work history by active resume version
4. Sort by date (most recent first)
5. Dispatch Redux actions

### Job Persistence Helpers

| Function | Purpose |
|----------|---------|
| `storeJobId(jobId)` | Save job ID + timestamp to localStorage |
| `getStoredJobId()` | Retrieve stored job ID (ignore if > 24 hours old) |
| `clearStoredJobId()` | Remove stored job ID |

### Student Parsing Helpers

**Source:** `src/helper/parsing/studentParsingHelpers.ts`

`fetchAndLoadStudentProfessionalData()` - mirrors the freelancer version but queries `student` schema tables.

## Environment Variables

The marketplace app requires these variables for parser integration:

```
AZURE_LONG_PARSER_URL=https://af-dv-resume-parser-{env}.azurewebsites.net/api/parse/long
AZURE_SHORT_PARSER_URL=https://af-dv-resume-parser-{env}.azurewebsites.net/api/parse/short
AZURE_API_KEY=<api-key-for-azure-functions>
NEXT_PUBLIC_BUCKET_NAME=<supabase-storage-bucket>
NEXT_PUBLIC_BUCKET_URL=https://<project>.supabase.co/storage/v1/object/public/<bucket>/
NEXT_PUBLIC_SERVER=development|test|production
```
