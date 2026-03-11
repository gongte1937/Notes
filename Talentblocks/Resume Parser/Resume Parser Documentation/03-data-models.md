# Data Models

Type definitions for the resume parser API. All interfaces are defined in `src/models/` in the parser service. For how these types flow through the system, see [Parsing Pipeline](02-parsing-pipeline.md). For how parsed data maps to database tables, see [Database Schema](04-database-schema.md).

## Shared Models

**Source:** `src/models/SharedModels.ts`

```typescript
interface FileData {
  content: Buffer;    // Raw file content
  fileName: string;   // Original filename
  size: number;       // File size in bytes
  mimeType: string;   // MIME type (e.g., application/pdf)
}

interface OcrResult {
  extractedText: string;
  confidence?: number;
}
```

## Short Parse Models

**Source:** `src/models/ShortParseModels.ts`

### ShortParseRequest

```typescript
interface ShortParseRequest {
  url: string;                              // Supabase storage URL
  fileType: 'txt' | 'md' | 'pdf' | 'docx'; // File format
  env: string;                              // 'development' | 'test' | 'production'
  encrypted: boolean;                        // CipherStash encryption flag
}
```

### ShortParseData

```typescript
interface ShortParseData {
  firstName?: string;      // e.g., "John"
  lastName?: string;       // e.g., "Doe"
  address?: string;        // Street address
  apartmentSuite?: string; // Apartment/suite number
  city?: string;           // e.g., "Sydney"
  zipCode?: number;        // e.g., 2000 (defaults to 0 if unknown)
  stateProvince?: string;  // e.g., "NSW"
  country?: string;        // e.g., "Australia"
  dateOfBirth?: string;    // ISO date
  mobileNumber?: string;   // Normalized: +61123456789 or 0123456789
  gender?: string;         // Only if explicitly stated in resume
}
```

### ShortParseResult

```typescript
interface ShortParseResult {
  success: boolean;
  data?: ShortParseData;
  error?: string;
  processingTime?: number;  // Milliseconds
  processedAt: Date;
}
```

## Long Parse Models

**Source:** `src/models/LongParseModels.ts`

### LongParseRequest

```typescript
interface LongParseRequest {
  url: string;                              // Supabase storage URL
  fileType: 'txt' | 'md' | 'pdf' | 'docx'; // File format
  env: string;                              // 'development' | 'test' | 'production'
  encrypted: boolean;                        // CipherStash encryption flag
  userId: string;                            // UUID (v1-v5) - required for DB storage
  versionName?: string;                      // Label for this resume version
  jobId?: string;                            // Links result to a parsing job
  resumeVersionId?: number;                  // Required - ID of resume_version created by Next.js
  isStudent?: boolean;                       // Routes data to student tables
}
```

### LongParseData

```typescript
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
    skill_name?: string;          // Normalized lowercase (e.g., "python")
    skill_category?: string;      // e.g., "Programming", "Cloud"
    proficiency_level?: string;   // beginner | intermediate | advanced | expert
    years_of_experience?: number; // Integer years
  }>;

  work_history?: Array<{
    job_title?: string;           // e.g., "Senior Software Engineer"
    company?: string;             // e.g., "Tech Corp"
    responsibilities?: string[];  // List of responsibility descriptions
    achievements?: string[];      // List of achievement descriptions
    date_start?: string;          // ISO date (YYYY-MM-DD)
    date_end?: string;            // ISO date or empty string for current
  }>;

  education_history?: Array<{
    institution?: string;         // e.g., "University of Melbourne"
    qualification?: string;       // e.g., "Bachelor of Science"
    field_of_study?: string;      // e.g., "Computer Science"
    date_start?: string;          // ISO date
    date_end?: string;            // ISO date or null for current
    grade?: string;               // e.g., "First Class Honours", "GPA 6.5"
  }>;

  certifications?: Array<{
    certification_name?: string;      // e.g., "AWS Solutions Architect"
    issuing_organization?: string;    // e.g., "Amazon Web Services"
    date_obtained?: string;           // ISO date
    expiry_date?: string;             // ISO date
    verification_link?: string;       // URL
  }>;

  projects?: Array<{
    project_client?: string;          // Client name
    project_name?: string;            // Project title
    project_description?: string;     // Description
    technology_used?: string[];       // Tech stack
    project_start?: string;           // ISO date
    project_end?: string;             // ISO date
    project_role?: string;            // Role in project
    project_responsibilities?: string[]; // Responsibilities
  }>;
}
```

### LongParseResult

```typescript
interface LongParseResult {
  success: boolean;
  data?: LongParseData;
  error?: string;
  processingTime?: number;          // Milliseconds
  processedAt: Date;
  storageSuccess?: boolean;          // Supabase storage outcome
  storageError?: string;
  jobUpdateSuccess?: boolean;        // parsing_jobs table update outcome
  jobUpdateError?: string;
  studentStorageSuccess?: boolean;   // Student table write outcome
  studentStorageError?: string;
}
```

## Internal Models

### StorageResult (Database Processor)

```typescript
interface StorageResult {
  success: boolean;
  error?: string;
  jobUpdateSuccess?: boolean;
  jobUpdateError?: string;
  studentStorageSuccess?: boolean;
  studentStorageError?: string;
  freelanceSyncSuccess?: boolean;
  freelanceSyncError?: string;
}
```

### ParseType (AI Processor)

```typescript
type ParseType = 'short' | 'long' | 'skills';
```

## Marketplace App Models

The marketplace app uses its own request/response shapes when calling the Azure Functions. See [Marketplace Integration](05-marketplace-integration.md) for the API routes that send these requests.

### Short Parse (Marketplace → Azure)

```typescript
// Request body sent by /api/parse/short
{
  url: fileUrl,        // Supabase storage URL
  fileType: string,    // pdf, docx, txt, md
  env: string,         // from NEXT_PUBLIC_SERVER
  encrypted: false,
  isStudent: boolean
}

// Response consumed by marketplace app
{
  personal: {
    firstName, lastName, email, mobileNumber,
    address, apartmentSuite, city, stateProvince,
    country, zipCode, dateOfBirth, gender
  }
}
```

### Long Parse Start (Marketplace → Azure)

```typescript
// Request body sent by /api/parse/long/start
{
  url: fileUrl,
  fileType: string,
  env: string,
  encrypted: false,
  userId: string,
  jobId: string,          // UUID generated by marketplace
  resumeVersionId: number, // Created by marketplace before calling
  isStudent: boolean
}

// Response: { jobId: string }
```

### Long Parse Status (Marketplace → Client)

```typescript
// Response from /api/parse/long/status?jobId=xxx
{
  status: 'pending' | 'running' | 'completed' | 'completed_with_errors' | 'failed' | 'read',
  error?: string,
  user_id: string,
  data?: LongParseResult  // Present when status is completed/completed_with_errors
}
```

## AI Prompt Output Formats

### Short Parse Prompt → JSON

```json
{
  "firstName": "",
  "lastName": "",
  "address": "",
  "apartmentSuite": "",
  "city": "",
  "zipCode": 0,
  "stateProvince": "",
  "country": "",
  "dateOfBirth": "",
  "mobileNumber": "",
  "gender": ""
}
```

### Long Parse Prompt → JSON

```json
{
  "work_history": [
    { "job_title": "", "company": "", "date_start": "", "date_end": "" }
  ],
  "education_history": [
    { "institution": "", "qualification": "", "field_of_study": "",
      "date_start": "", "date_end": "", "grade": "" }
  ]
}
```

### Skills Parse Prompt → CSV

```csv
skill_name,skill_category,proficiency_level,years_of_experience
python,programming language,advanced,5
sql,programming language,,3
excel,tool,intermediate,
```
