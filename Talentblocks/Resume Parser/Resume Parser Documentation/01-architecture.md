# Architecture

## System Overview

The resume parser is a distributed system with three main layers:

```
                         ┌─────────────────────────────────┐
                         │     Marketplace App (Next.js)   │
                         │                                 │
                         │  ┌─────────┐  ┌──────────────┐  │
                         │  │Frontend │  │  API Routes  │  │
                         │  │(React)  │──│ /api/parse/* │  │
                         │  └─────────┘  └──────┬───────┘  │
                         └──────────────────────┼──────────┘
                                                │
                                    HTTP (X-API-Key)
                                                │
                         ┌──────────────────────▼────────────┐
                         │   Resume Parser (Azure Functions) │
                         │                                   │
                         │  ┌──────────┐  ┌──────────────┐   │
                         │  │  Short   │  │    Long      │   │
                         │  │  Parser  │  │    Parser    │   │
                         │  └────┬─────┘  └──────┬───────┘   │
                         │       │               │           │
                         │  ┌────▼───────────────▼─────────┐ │
                         │  │       Processing Pipeline    │ │
                         │  │  File Fetch → OCR → AI → DB  │ │
                         │  └──────────────────────────────┘ │
                         └──────────┬──────────┬─────────────┘
                                    │          │
                    ┌───────────────┘          └──────────────┐
                    ▼                                          ▼
          ┌─────────────────┐                      ┌──────────────────┐
          │  Azure OpenAI   │                      │    Supabase      │
          │  (GPT-5-mini)   │                      │  (PostgreSQL)    │
          │                 │                      │                  │
          │  - Short parse  │                      │ - resume schema  │
          │  - Long parse   │                      │ - student schema │
          │  - Skills parse │                      │ - public schema  │
          └─────────────────┘                      └──────────────────┘
```

## Service Dependencies

### Azure Functions Runtime
- **Runtime:** Node.js 20.x
- **Framework:** Azure Functions v4
- **Timeout:** 2 minutes (configured in `host.json`)
- **Auth Level:** Anonymous (application code validates the API key)

### Azure OpenAI
- **Deployment:** `gpt-5-mini`
- **API Version:** `2025-01-01-preview`
- **Usage:** Three prompt types for extraction tasks
  - `SHORT_PARSE` - Contact information only
  - `LONG_PARSE` - Work history, education, certifications, projects (JSON output)
  - `LONG_SKILLS_PARSE` - Skills extraction and normalization (CSV output)
- **Retry:** 3 attempts with exponential backoff (5s base, 1.5x multiplier)

### Azure Document Intelligence
- **Model:** `prebuilt-read`
- **Usage:** OCR for PDF and DOCX files
- **Pattern:** Async polling (`beginAnalyzeDocument()` → `pollUntilDone()`)
- **Skipped for:** TXT and MD files (direct UTF-8 read)

### Azure Key Vault
- **Purpose:** Centralized secret storage
- **Caching:** In-memory cache per function invocation minimizes API calls
- **Secrets:** See [Security & Configuration](07-security-and-config.md)

### S3 Functions (File Storage Service)
- **Repository:** `pvt-talentblocks-supabase-storage`
- **URL:** `https://af-dv-s3-functions.azurewebsites.net`
- **Route:** `POST /api/read-file-s3`
- **Purpose:** Reads files from Supabase Storage with optional CipherStash decryption
- **Auth:** `x-functions-key` header

### Supabase
- **Type:** PostgreSQL with RLS
- **Schemas:** `resume` (freelancer data), `student` (student data), `public` (user validation)
- **Client:** `@supabase/supabase-js` with service role key

## Environment Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        Development                              │
│                                                                 │
│  Azure Function:  af-dv-resume-parser-dev.azurewebsites.net     │
│  Supabase:        idquneahfcyfkoujwhia                          │
│  Key Vault:       (dev vault)                                   │
│  S3 Functions:    af-dv-s3-functions.azurewebsites.net          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                           Test                                  │
│                                                                 │
│  Azure Function:  af-dv-resume-parser-test.azurewebsites.net    │
│  Supabase:        oelmeeycrttqddjumvga                          │
│  Key Vault:       (test vault)                                  │
│  S3 Functions:    af-dv-s3-functions.azurewebsites.net (shared) │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        Production                               │
│                                                                 │
│  Azure Function:  af-dv-resume-parser-prod.azurewebsites.net    │
│  Supabase:        zppdrquirnteyhdxaokk                          │
│  Key Vault:       (prod vault)                                  │
│  S3 Functions:    af-dv-s3-functions.azurewebsites.net (shared) │
└─────────────────────────────────────────────────────────────────┘
```

All environments share the S3 Functions service. The `env` parameter in each request routes to the correct Supabase storage bucket.

## Project Structure (Parser Service)

```
pvt-talentblocks-resume-parser/
├── src/
│   ├── index.ts                          # Entry point (ping endpoint)
│   ├── functions/
│   │   ├── httpTriggers.ts               # Short & long parse HTTP endpoints
│   │   ├── processors/
│   │   │   ├── fileProcessor.ts          # S3 file reading, OCR, text extraction
│   │   │   ├── aiProcessor.ts            # OpenAI calls, JSON/CSV cleanup
│   │   │   └── databaseProcessor.ts      # Supabase storage logic
│   │   ├── validators/
│   │   │   └── requestValidator.ts       # API key & request validation
│   │   └── utils/
│   │       └── dateUtils.ts              # ISO date parsing & validation
│   ├── models/
│   │   ├── SharedModels.ts               # FileData, OcrResult interfaces
│   │   ├── ShortParseModels.ts           # Short parse types
│   │   └── LongParseModels.ts            # Long parse types
│   ├── prompts/
│   │   └── SystemPrompts.ts              # AI system prompts (3 types)
│   └── services/
│       ├── KeyVaultService.ts            # Azure Key Vault client with caching
│       └── HttpService.ts               # Axios HTTP client wrapper
├── tests/
│   ├── unit/                             # Jest unit tests (90% coverage)
│   ├── helpers/                          # Mock context, test utilities
│   └── setup/                            # Global test configuration
├── .github/workflows/
│   ├── deploy-dev.yml                    # Dev CI/CD
│   └── deploy-test.yml                   # Test CI/CD
├── host.json                             # Azure Functions configuration
├── package.json                          # Dependencies
└── jest.config.js                        # Test configuration
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20.x |
| Language | TypeScript 4.x |
| Serverless | Azure Functions v4 |
| AI | Azure OpenAI (GPT-5-mini) |
| OCR | Azure Document Intelligence |
| Database | Supabase (PostgreSQL) |
| File Storage | Supabase Storage via S3 Functions |
| Secrets | Azure Key Vault |
| HTTP Client | Axios |
| PDF Parsing | pdf-parse |
| DOCX Parsing | mammoth |
| MIME Detection | mimetics |
| Testing | Jest with ts-jest |
| CI/CD | GitHub Actions |
