# Resume Parser Migration Plan: Azure Functions -> NestJS (Dual Textkernel Endpoints)

## 1. Background and Decision

The current flow uses Marketplace -> Azure Functions -> Supabase. Based on team discussion, the migration plan is now:

1. NestJS calls **Textkernel Parse Resume endpoint**.
2. NestJS calls **Textkernel Extract Skills endpoint**.
3. Return parsed basic profile fields from Resume Parse to frontend first.
4. Then run skills mapping from Skills Extract result and write to DB.

> Core principle: frontend gets basic parsed data first, then skills + DB persistence complete as a follow-up step; isolate changes in parser integration and mapping.

---

## 2. Target Architecture

```text
User uploads resume PDF/DOCX
  └──► S3 upload function
        └──► Supabase Storage
              └──► Marketplace API
                    └──► NestJS Resume Parser
                          ├──► Textkernel Parse Resume endpoint
                          ├──► Basic Info Adapter (fill frontend fields)
                          ├──► Textkernel Extract Skills endpoint
                          ├──► MappingService (skills mapping)
                          └──► DatabaseService (persist to Supabase)
```

---

## 3. Post-Migration Flow (To-Be)

1. User uploads resume (recommended input restriction: `pdf` / `docx`).
2. File is stored in Supabase Storage through the existing S3 upload function.
3. Marketplace calls NestJS parser API (keep current auth contract).
4. NestJS reads file/text from storage.
5. NestJS calls Textkernel `parse resume` endpoint for primary structured data.
6. Basic fields are extracted and returned to frontend (name, contacts, core education/experience fields).
7. NestJS calls Textkernel `extract skills` endpoint for skills payload.
8. `MappingService` normalizes and maps skills data.
9. `DatabaseService` writes to DB using existing storage rules.
10. Frontend keeps existing polling flow until completion.

---

## 4. API and Data Handling Design

### 1) Dual Textkernel Endpoint Calls

| Use case | Endpoint | Purpose | Output target |
| --- | --- | --- | --- |
| Main parsing | `parse resume` | Work history, education, personal profile | Frontend basic profile fields |
| Skills parsing | `extract skills` | Skill names, proficiency, experience | Skills mapping + skills-related DB tables |

### 2) Merge and Mapping Strategy

- Execute in sequence: `parse resume` first, then `extract skills`.
- `parse resume` output is used for frontend basic info population; legacy intermediate parse structures are no longer produced.
- `extract skills` output goes through skills mapping (naming, level, experience normalization).
- Write mapping anomalies to diagnostics logs (no silent failures).

### 3) Database Strategy

- Keep current DB schema unchanged.
- Keep existing storage behaviors unchanged (optimistic lock, student authorization, non-blocking subsidiary inserts).
- DB write is triggered after skills mapping is complete.
- Major changes are in endpoint orchestration and `MappingService`, not in schema design.

---

## 5. NestJS Modules and Responsibilities

### Modules

- `AppModule`
- `ParseModule`
- `HealthModule`
- `ConfigModule`
- `HttpModule`

### Controllers

- `ParseController`
- `HealthController`

### Services

- `ParseService`: orchestration
- `TextkernelResumeService`: parse resume endpoint integration + extraction of frontend basic fields
- `FrontendPayloadService`: shape and return frontend basic info payload
- `TextkernelSkillsService`: extract skills endpoint integration
- `MappingService`: skills mapping and normalization
- `DatabaseService`: Supabase persistence
- `FileService`: file/text retrieval
- `ApiKeyAuthService`: authentication

---

## 6. Work Breakdown (from team discussion)

### 1) Research story

| Task | Owner |
| --- | --- |
| Test API calls (validate both Textkernel endpoints) | Terry |

### 2) Textkernel API development and changes

| Task | Owner |
| --- | --- |
| Setup API call to parse a resume | Terry |
| Parse result to frontend basic info payload | Terry |
| Setup API call to extract skills from parsed resume text | Terry |
| Skills mapping and DB write integration | Terry |

### 3) Testing

| Task | Owner |
| --- | --- |
| Unit tests | Terry |
| Integration tests | Terry |
| E2E tests | Terry |
| UAT | Stefan |

### 4) Cleanup

| Task | Owner |
| --- | --- |
| Remove existing resume parser calls to Azure Function | Terry |
| Remove existing Azure Function infrastructure | Shane |
| Create a commercial account (Textkernel) | Shane |

---

## 7. Phase Plan

### Phase 1: API validation and minimum viable flow

- [ ] Verify both Textkernel endpoints with real samples.
- [ ] Validate response completeness and quality.
- [ ] Complete parse resume -> frontend basic info population.
- [ ] Draft skills mapping rules.

### Phase 2: Backend integration

- [ ] Add `TextkernelResumeService` and `TextkernelSkillsService` in NestJS.
- [ ] Add `FrontendPayloadService` to return basic info.
- [ ] Implement skills `MappingService` and connect to existing `DatabaseService` writes.

### Phase 3: Testing and UAT

- [ ] Unit tests for mapping, fallback, and error handling.
- [ ] Integration tests for dual-endpoint + DB flow.
- [ ] E2E tests for upload -> parse -> save -> display.
- [ ] UAT sign-off.

### Phase 4: Cutover and decommission

- [ ] Remove Marketplace calls to old Azure Function parser.
- [ ] Roll out by environment (dev -> test -> prod).
- [ ] After stability window, decommission old Azure Function infrastructure.

---

## 8. Key Risks and Controls

1. Inconsistent payloads between two endpoints
- Control: define strict responsibility boundaries (parse for basic profile, skills endpoint for skill domain).

2. Type mismatches (dates, experience, proficiency)
- Control: normalize in mapping layer and cover with contract tests.

3. Missing/unstable skills extraction
- Control: fallback strategy (empty array/default values) + diagnostics logging.

4. Performance and timeout
- Control: explicit timeout/retry per endpoint; prioritize returning basic frontend info and complete skills + DB writes as follow-up.

---

## 9. Open Questions

1. Do we need to keep original resume PDFs long-term (compliance, cost, traceability)?
2. What is the procurement/activation timeline for Textkernel commercial account?
3. If skills endpoint fails, should main parse still complete with a partial-success state?
4. After frontend basic info is populated, how should UI represent skills/DB write status (in-progress/partial success)?

---

## 10. Rollback Plan

Keep legacy flow available during migration. If issues occur:

1. Switch Marketplace parser URL back to Azure Function.
2. Redeploy or hot-apply env changes.
3. Legacy pipeline resumes traffic immediately.
