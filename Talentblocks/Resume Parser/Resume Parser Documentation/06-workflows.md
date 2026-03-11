# Workflows

End-to-end user workflows for resume parsing, from upload through data consumption. For API route details, see [Marketplace Integration](05-marketplace-integration.md). For database tables involved, see [Database Schema](04-database-schema.md).

## Onboarding Flow (First Resume)

When a freelancer uploads their first resume during account setup, both parsers run simultaneously:

```
User selects resume file (PDF/DOCX)
│
├──► Client computes SHA-256 hash
├──► Upload to Supabase Storage (contractorResume/{timestamp}-{filename})
│
├──► POST /api/parse/short (synchronous)
│     │
│     ├── Validate auth + user ownership
│     ├── Call Azure Short Parser
│     └── Return: { personal: { firstName, lastName, ... } }
│           │
│           ├──► Encrypt personal details via /api/encrypt-freelancer-data
│           ├──► Save to freelancer_personal_details (upsert)
│           └──► Dispatch Redux: setUserPersonalDetail()
│
└──► POST /api/parse/long/start (asynchronous)
      │
      ├── Cancel existing pending/running jobs
      ├── Create parsing_jobs record (status: "running")
      ├── Deactivate all existing resume_versions
      ├── Create new resume_versions record (is_active: true)
      ├── Fire processResumeAsync() (background)
      └── Return: { jobId }
            │
            ├──► Store jobId in localStorage
            ├──► Start progress bar animation (5% → 60% → 95%)
            │
            └──► Poll GET /api/parse/long/status?jobId=xxx (every 4s)
                  │
                  ├── status: "running" → continue polling
                  ├── status: "completed" → stop polling
                  │     │
                  │     ├──► fetchAndLoadProfessionalData()
                  │     │     ├── Query resume_work_experience
                  │     │     ├── Query skills
                  │     │     └── Query professional_details
                  │     │
                  │     ├──► Dispatch Redux: setWorkHistory()
                  │     ├──► Dispatch Redux: setUserSkills()
                  │     ├──► Dispatch Redux: setUserProfessionalDetail()
                  │     ├──► Dispatch Redux: setParserFinalData()
                  │     └──► Progress bar: 100% → reset after 2s
                  │
                  ├── status: "completed_with_errors" → show warning
                  └── status: "failed" → show error
```

## Update Resume Flow

When a freelancer uploads a new resume to replace their current one:

```
User uploads new resume file
│
├──► Upload to Supabase Storage
│
└──► POST /api/parse/long/start
      │
      ├── Cancel existing pending/running jobs
      ├── Deactivate ALL resume_versions for user (is_active: false)
      ├── Create new resume_versions record (is_active: true)
      ├── Create parsing_jobs record (status: "running")
      ├── Start background parsing
      └── Return: { jobId }
            │
            └──► Poll for completion (same as onboarding)
                  │
                  └── On completion:
                        ├──► Fetch professional data filtered by active resume
                        ├──► Update Redux state
                        └──► Refresh resume list
```

**Key difference from onboarding:** The update flow skips the short parse and runs only the long parser, since personal details already exist.

## Resume Version Management

### Activate a Resume Version

```
User selects "Make Active" on an inactive resume
│
├──► Set is_active=false for ALL resume_versions
├──► Set is_active=true for selected version
│
└──► Refresh freelancer work history
      └── Associates work_history entries with the newly active resume
```

### Delete a Resume Version

```
User selects "Delete" on an inactive resume
│
├──► Confirm deletion dialog
│
└──► Set is_deleted=true (soft delete)
      └── Resume no longer appears in list
```

**Constraints:**
- The active resume cannot be deleted
- Soft deletion sets `is_deleted=true` and preserves data

## Job Recovery (Page Refresh)

If the user refreshes the page during parsing, the app recovers the job from localStorage.

```
Page loads
│
├──► Check localStorage for currentParsingJobId
│     │
│     ├── No stored ID → normal page load
│     │
│     └── Found stored ID
│           │
│           ├── Check age: > 24 hours? → clear and ignore
│           │
│           └── Poll /api/parse/long/status?jobId=xxx
│                 │
│                 ├── Job found + completed → load data
│                 ├── Job found + running → resume polling
│                 ├── Job found + failed → show error, clear storage
│                 └── Job not found (404) → clear storage silently
```

**localStorage keys:**
- `currentParsingJobId` - The job UUID
- `currentParsingJobStartedAt` - Timestamp for age validation

## Student Parsing Flow

Mirrors the freelancer flow but passes the `isStudent: true` flag.

```
Student uploads resume
│
├──► POST /api/parse/short (same as freelancer)
│     └── Save to student personal details
│
└──► POST /api/parse/long/start (with isStudent: true)
      │
      └── Azure Function processes resume
            │
            ├── Write to freelancer tables (resume schema)
            │
            └── Write to student tables (student schema)
                  │
                  ├── Validate users.userType === 'studentUser'
                  │     │
                  │     ├── Authorized → write to student tables
                  │     └── Not authorized → skip, set studentStorageSuccess=false
                  │
                  ├── student_work_history  ← INSERT
                  ├── student_education_history ← INSERT
                  └── student_skills ← INSERT
```

**Student tables receive the same data as freelancer tables** with minor schema differences: student work history omits responsibilities/achievements, and student education omits certificate_url. See [Database Schema — Student Tables](04-database-schema.md#student-schema-tables) for column details.

## Concurrent Job Prevention

Each user can run only one parsing job at a time.

```
New parse request arrives for user X
│
├──► Query parsing_jobs WHERE user_id=X AND status IN ('pending','running')
│
├── Found existing jobs?
│     │
│     ├── Yes → Cancel all existing jobs (status: 'cancelled')
│     └── No → Continue
│
└──► Create new job (status: 'running')
```

This prevents duplicate processing and ensures the system parses only the latest upload.

## Data Flow Summary

```
                                  ┌──────────────────────┐
                                  │   User uploads file  │
                                  └──────────┬───────────┘
                                             │
                              ┌──────────────┼──────────────┐
                              ▼              ▼              │
                       ┌────────────┐  ┌────────────┐       │
                       │ Short Parse│  │ Long Parse │       │
                       │ (sync)     │  │ (async)    │       │
                       └─────┬──────┘  └─────┬──────┘       │
                             │               │              │
                             ▼               ▼              │
                    ┌──────────────┐  ┌────────────────┐    │
                    │ Personal     │  │ Resume Schema  │    │
                    │ Details      │  │ Tables         │    │
                    │ (encrypted)  │  ├────────────────┤    │
                    └──────────────┘  │ Public Schema  │    │
                                      │ Tables         │    │
                                      ├────────────────┤    │
                                      │ Student Schema │    │
                                      │ (if student)   │    │
                                      └───────┬────────┘    │
                                              │             │
                                              ▼             │
                                      ┌──────────────┐      │
                                      │ Redux Store  │      │
                                      │ (UI state)   │      │
                                      └──────────────┘      │
                                                            │
                                      ┌──────────────┐      │
                                      │ Supabase     │◄─────┘
                                      │ Storage      │
                                      └──────────────┘
```
