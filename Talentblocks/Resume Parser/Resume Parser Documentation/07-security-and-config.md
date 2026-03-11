# Security & Configuration

## Authentication

For the system architecture and service topology, see [Architecture](01-architecture.md). For where validation occurs in the processing flow, see [Parsing Pipeline — Step 1](02-parsing-pipeline.md#step-1-request-validation).

### Parser Service (Azure Functions)

**API Key Authentication:**
- Every request requires `X-API-Key` header
- Key stored in Azure Key Vault as `resume-parser-api-key`
- Cached in-memory per function invocation to eliminate repeated Key Vault calls
- Returns `401` for missing/invalid key, `503` if Key Vault is unavailable

### Marketplace App (Next.js API Routes)

**Bearer Token Authentication:**
- All parse routes require `Authorization: Bearer <token>` header
- Supabase Auth validates the token (`userSupabase.auth.getUser()`)
- The route creates a user-context Supabase client with anon key + user token
- Ownership check: `users.auth_user_id` must match the authenticated user

```
Client ──Bearer token──► Next.js API Route ──X-API-Key──► Azure Function
          (Supabase Auth)                      (Key Vault)
```

### User Ownership Validation

Both the marketplace app and parser service validate ownership:

**Marketplace app** checks `users.auth_user_id` matches the authenticated Supabase user.

**Parser service** checks `resume_versions.user_id` matches the `userId` parameter when updating resume data.

### Student Authorization

When `isStudent: true`, the parser validates the user is authorized to write to student tables:

```typescript
const { data: userData } = await publicSupabase
  .from("users")
  .select("userType")
  .eq("id", userId)
  .single();

// Must be 'studentUser' to write to student tables
if (userData.userType !== "studentUser") {
  // Skip student tables, log warning
}
```

## Encryption

### CipherStash

Supabase Storage can encrypt resume files with CipherStash. The `encrypted` flag in parse requests tells the S3 Functions service to decrypt the file before returning it.

**Encrypted fields in Supabase** (freelancer personal details):
- `enc_address`
- `enc_zip`
- `enc_suburb`
- `enc_apartment`

The short parser extracts personal details, then `/api/encrypt-freelancer-data` encrypts them before storage.

### Data in Transit

- Marketplace ↔ Azure Functions: HTTPS
- Azure Functions ↔ Azure OpenAI: HTTPS
- Azure Functions ↔ Azure Key Vault: HTTPS with managed identity
- Azure Functions ↔ Supabase: HTTPS with service role key
- Azure Functions ↔ S3 Functions: HTTPS with function key

## Azure Key Vault

### Secrets Inventory

| Secret Name | Purpose | Used By |
|-------------|---------|---------|
| `document-intelligence-endpoint` | Azure Document Intelligence URL | fileProcessor |
| `document-intelligence-key` | Azure Document Intelligence API key | fileProcessor |
| `openai-endpoint` | Azure OpenAI base URL | aiProcessor |
| `openai-key` | Azure OpenAI API key | aiProcessor |
| `supabase-url` | Supabase project URL | databaseProcessor |
| `supabase-service-key` | Supabase service role key | databaseProcessor |
| `s3-function-key` | S3 Functions API key | fileProcessor |
| `resume-parser-api-key` | Parser API key for X-API-Key validation | requestValidator |
| `parser-env` | Environment name | requestValidator |

### Caching Strategy

`KeyVaultService` caches secrets in-memory within a single function invocation:

```typescript
class KeyVaultService {
  private cache: Map<string, string> = new Map();

  async getSecret(name: string, context: InvocationContext): Promise<string> {
    if (this.cache.has(name)) {
      context.log(`Key Vault cache hit: ${name}`);
      return this.cache.get(name)!;
    }
    // Fetch from Azure Key Vault, cache result
  }
}
```

The cache is per-instance. Azure Functions may reuse instances across invocations, so the cache can persist across requests on the same instance.

### Access Control

The Azure Function App authenticates to Key Vault with `DefaultAzureCredential`, which uses managed identity in Azure and developer credentials locally.

## Environment Variables

### Parser Service (Azure Functions)

Set via Azure Function App configuration, not `.env` files:

| Variable | Purpose |
|----------|---------|
| `KEY_VAULT_URL` | Azure Key Vault URL |
| `S3_FUNCTION_URL` | S3 Functions base URL (`https://af-dv-s3-functions.azurewebsites.net`) |

Key Vault secrets supply all other configuration.

### Marketplace App (Next.js)

Required in `.env.local`:

| Variable | Purpose |
|----------|---------|
| `AZURE_LONG_PARSER_URL` | Azure Function long parse endpoint |
| `AZURE_SHORT_PARSER_URL` | Azure Function short parse endpoint |
| `AZURE_API_KEY` | API key for Azure Functions |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anonymous key |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key |
| `NEXT_PUBLIC_BUCKET_NAME` | Supabase Storage bucket name |
| `NEXT_PUBLIC_BUCKET_URL` | Supabase Storage public bucket URL |
| `NEXT_PUBLIC_SERVER` | Environment name (`development`, `test`, `production`) |

## Azure Functions Configuration

### `host.json` (see [Deployment & Testing](08-deployment-and-testing.md) for CI/CD and environment details)

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensions": {
    "http": {
      "routePrefix": "api"
    }
  },
  "functionTimeout": "00:02:00"
}
```

Key settings:
- **Function timeout:** 2 minutes (sufficient for OCR + AI processing)
- **Application Insights sampling:** Enabled, excludes Request type
- **Route prefix:** `api` (endpoints are `/api/parse/short`, `/api/parse/long`)

### HTTP Trigger Registration

```typescript
// Anonymous auth level - API key validated in application code
app.http("ShortResumeParser", {
  methods: ["POST"],
  authLevel: "anonymous",
  route: "parse/short",
  handler: shortResumeParser,
});

app.http("LongResumeParser", {
  methods: ["POST"],
  authLevel: "anonymous",
  route: "parse/long",
  handler: longResumeParser,
});
```

Auth level is `anonymous` because application code validates the API key directly, bypassing Azure Functions built-in auth.

## Security Considerations

1. **No direct user impersonation:** The parser uses a service role key for database access, validated against the userId parameter
2. **Optimistic locking:** Prevents concurrent modification of resume data
3. **Soft deletes:** The system retains all resume versions (no hard deletes)
4. **Non-blocking errors:** Subsidiary table write failures preserve data integrity
5. **Input validation:** UUID format validation, file type whitelist, environment whitelist
6. **Temp file cleanup:** `finally` blocks delete temporary files created during OCR
