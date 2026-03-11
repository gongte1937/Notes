# Deployment & Testing

For environment topology and service URLs, see [Architecture](01-architecture.md). For secrets and environment variables, see [Security & Configuration](07-security-and-config.md). For production debugging, see [Troubleshooting](09-troubleshooting.md).

## CI/CD Pipelines

### GitHub Actions Workflows

Two workflows handle automated deployment:

#### Development (`deploy-dev.yml`)

**Trigger:** Push to `development` branch or manual `workflow_dispatch`

```
┌──────────────┐     ┌──────────────────────┐
│  Test Job    │────►│  Build & Deploy Job  │
│              │     │                      │
│ - Checkout   │     │ - Checkout           │
│ - Node 20.x  │     │ - Node 20.x          │
│ - npm ci     │     │ - npm ci             │
│ - npm test   │     │ - npm run build (tsc)│
│              │     │ - Azure login        │
│              │     │ - Deploy to Function │
└──────────────┘     └──────────────────────┘
```

- Test job must pass before deploy
- Node.js 20.x with npm cache
- Deploys to `af-dv-resume-parser-dev`
- Uses `AZURE_CREDENTIALS_DEV` GitHub secret

#### Test (`deploy-test.yml`)

Same structure as dev, deploys to `af-dv-resume-parser-test`.

#### Production

Production deployment follows the same pattern, deploying to `af-dv-resume-parser-prod`. A push to the `production` branch triggers it.

### Concurrency

GitHub Actions concurrency settings prevent simultaneous deployments:

```yaml
concurrency:
  group: deploy-dev
  cancel-in-progress: true
```

### Build Process

```bash
npm run clean    # Remove dist/ folder
tsc              # Compile TypeScript to dist/
```

Output structure:
```
dist/
├── index.js
├── functions/
│   ├── httpTriggers.js
│   ├── processors/
│   ├── validators/
│   └── utils/
├── models/
├── prompts/
└── services/
```

## Azure Function Apps

| Environment | Function App Name | URL |
|-------------|-------------------|-----|
| Development | `af-dv-resume-parser-dev` | `https://af-dv-resume-parser-dev.azurewebsites.net` |
| Test | `af-dv-resume-parser-test` | `https://af-dv-resume-parser-test.azurewebsites.net` |
| Production | `af-dv-resume-parser-prod` | `https://af-dv-resume-parser-prod.azurewebsites.net` |

### Health Check

```bash
curl https://af-dv-resume-parser-dev.azurewebsites.net/api/ping
# Response: "pong"
```

### Monitoring

- **Application Insights:** Integrated via Azure Functions runtime
- **Logging:** Structured logs via `InvocationContext` (log, debug, info, warn, error)
- **Metrics:** Processing time, file size, OpenAI latency, record counts

## Testing

### Framework

| Component | Tool |
|-----------|------|
| Test runner | Jest 30.x |
| TypeScript support | ts-jest 29.x |
| Mocking | jest-mock-extended |
| Coverage reporter | text, lcov, HTML |

### Configuration (`jest.config.js`)

```javascript
{
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/tests/**/*.test.ts'],
  coverageThreshold: {
    global: {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  }
}
```

**Coverage requirement:** 90% across all four metrics (branches, functions, lines, statements).

Coverage excludes models, interfaces, and `index.ts`.

### Test Structure

```
tests/
├── unit/
│   ├── processors/
│   │   ├── fileProcessor.test.ts     # S3 reading, OCR, text extraction
│   │   ├── aiProcessor.test.ts       # OpenAI calls, JSON/CSV cleanup
│   │   └── databaseProcessor.test.ts # Supabase storage logic
│   ├── validators/
│   │   └── requestValidator.test.ts  # API key + field validation
│   ├── services/
│   │   ├── KeyVaultService.test.ts   # Key Vault caching
│   │   └── HttpService.test.ts       # HTTP client wrapper
│   └── utils/
│       └── dateUtils.test.ts         # ISO date parsing
├── helpers/
│   ├── contextMock.ts                # InvocationContext mock
│   └── testUtils.ts                  # Test utilities
└── setup/
    └── setupTests.ts                 # Global configuration
```

### Mock Context

Tests use a mock `InvocationContext` factory:

```typescript
function createMockContext(overrides?: Partial<InvocationContext>) {
  return {
    log: jest.fn(),
    debug: jest.fn(),
    info: jest.fn(),
    warn: jest.fn(),
    error: jest.fn(),
    invocationId: 'test-invocation-id',
    functionName: 'test-function',
    ...overrides
  };
}
```

### Test Setup (`setupTests.ts`)

Runs before each test:
- Clears all mocks
- Resets environment variables
- Sets `NODE_ENV` to `test`
- Mocks `KEY_VAULT_URL` and `S3_FUNCTION_URL`
- Installs a global unhandled rejection handler

### Running Tests

```bash
# Run all tests
npm test

# Run tests with coverage
npm run test:coverage

# Run specific test file
npx jest tests/unit/processors/aiProcessor.test.ts

# Run with verbose output
npm test -- --verbose
```

### Test Timeout

Default Jest timeout: 10 seconds (sufficient for mocked async operations).

## Scripts

```json
{
  "build": "tsc",
  "clean": "rimraf dist",
  "start": "func start",
  "test": "jest",
  "test:coverage": "jest --coverage",
  "lint": "eslint src/"
}
```

## Local Development

### Prerequisites

- Node.js 20.x (via mise)
- Azure Functions Core Tools v4
- Azure CLI (for Key Vault access)

### Running Locally

```bash
# Install dependencies
npm ci

# Start the function app locally
npm start
# or
func start
```

The local function app authenticates to Azure Key Vault with developer credentials (`az login`).

### Testing Against Local

```bash
curl -X POST http://localhost:7071/api/parse/short \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-dev-api-key" \
  -d '{
    "url": "https://...",
    "fileType": "pdf",
    "env": "development",
    "encrypted": false
  }'
```
