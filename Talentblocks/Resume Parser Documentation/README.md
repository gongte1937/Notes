# Resume Parser System

The Resume Parser extracts structured data from uploaded resumes (PDF, DOCX, TXT, MD) using AI and stores the results in Supabase. It spans two repositories and three Azure services (Functions, OpenAI, Document Intelligence).

## Repositories

| Repository | Purpose | Tech |
|-----------|---------|------|
| `pvt-talentblocks-resume-parser` | Azure Functions service that performs OCR, AI extraction, and database storage | Node.js, TypeScript, Azure Functions |
| `pvt-talentblocks-marketplace-app` | Next.js app that triggers parsing, polls for results, and renders data | Next.js 14, React, Redux |

## How It Works

1. User uploads a resume file to Supabase Storage
2. Marketplace app calls the Azure Function parser via API routes
3. Parser extracts text (OCR for PDF/DOCX, direct read for TXT/MD)
4. Azure OpenAI (GPT-5-mini) extracts structured data in parallel (main content + skills)
5. Parser stores extracted data in Supabase tables (freelancer or student schema)
6. Marketplace app polls for completion and loads the results into Redux state

## Two Parsing Modes

| Mode | Endpoint | AI Model | Database Storage | Use Case |
|------|----------|----------|-----------------|----------|
| **Short Parse** | `POST /api/parse/short` | GPT-5-mini | No | Extract contact info (name, address, phone) |
| **Long Parse** | `POST /api/parse/long` | GPT-5-mini | Yes | Extract full career history, skills, education |

## Documentation

| Document | Content |
|----------|---------|
| [Architecture](01-architecture.md) | System architecture, service dependencies, environment topology |
| [Parsing Pipeline](02-parsing-pipeline.md) | End-to-end flow: validation, OCR, AI processing, database storage |
| [Data Models](03-data-models.md) | Request/response types, parsed data structures |
| [Database Schema](04-database-schema.md) | Tables, schemas, relationships, optimistic locking |
| [Marketplace Integration](05-marketplace-integration.md) | API routes, frontend components, Redux state, hooks |
| [Workflows](06-workflows.md) | Onboarding, resume update, version management, job recovery |
| [Security & Configuration](07-security-and-config.md) | Authentication, encryption, Key Vault, environment variables |
| [Deployment & Testing](08-deployment-and-testing.md) | CI/CD pipelines, environments, test framework |
| [Troubleshooting](09-troubleshooting.md) | Error codes, common issues, debugging guide |

## Quick Reference

### Azure Function Apps

| Environment | URL | Supabase Project |
|-------------|-----|-----------------|
| Development | `af-dv-resume-parser-dev.azurewebsites.net` | `idquneahfcyfkoujwhia` |
| Test | `af-dv-resume-parser-test.azurewebsites.net` | `oelmeeycrttqddjumvga` |
| Production | `af-dv-resume-parser-prod.azurewebsites.net` | `zppdrquirnteyhdxaokk` |

### Marketplace API Routes

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/parse/short` | POST | Synchronous contact info extraction |
| `/api/parse/long/start` | POST | Start async full parsing job |
| `/api/parse/long/status` | GET | Poll job status and retrieve results |
| `/api/parse/long/run-sync` | POST | Synchronous full parsing (testing/fallback) |

### Key Source Files

**Parser Service (`pvt-talentblocks-resume-parser`):**
- `src/functions/httpTriggers.ts` - HTTP endpoint definitions
- `src/functions/processors/fileProcessor.ts` - File reading, OCR, text extraction
- `src/functions/processors/aiProcessor.ts` - OpenAI calls, response cleaning
- `src/functions/processors/databaseProcessor.ts` - Supabase storage logic
- `src/prompts/SystemPrompts.ts` - AI system prompts

**Marketplace App (`pvt-talentblocks-marketplace-app`):**
- `src/app/api/parse/short/route.ts` - Short parse API route
- `src/app/api/parse/long/start/route.ts` - Long parse start API route
- `src/app/api/parse/long/status/route.ts` - Job status polling API route
- `src/app/freelancer/resume/page.tsx` - Freelancer resume upload page
- `src/helper/parsing/parsingHelpers.ts` - Parsing utility functions

Last Updated: 2026-03-03
