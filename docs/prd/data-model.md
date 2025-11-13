# Data Model

## Overview

The AI Ops Prompt IDE uses a file-based data model with JSON as the primary serialization format. All execution data is stored in timestamped folders for auditability and reproducibility. This document defines the schemas for all data structures used within the system.

## Core Entities

### 1. Execution Context

Represents a single execution run of the comparison tool.

```typescript
interface ExecutionContext {
  executionId: string;           // UUID v4
  timestamp: string;              // ISO 8601 format
  lotId: string;                  // HiBid lot identifier
  lotUrl: string;                 // Original HiBid URL
  models: ModelConfig[];          // Models configured for this execution
  promptTemplate: string;         // Prompt template used
  executionFolder: string;        // Absolute path to execution folder
  status: ExecutionStatus;        // current, completed, failed, partial
  errorMessage?: string;          // Present if status is failed
  duration: number;               // Total execution time in milliseconds
}

enum ExecutionStatus {
  CURRENT = "current",
  COMPLETED = "completed",
  FAILED = "failed",
  PARTIAL = "partial"
}
```

**File Location**: `executions/{timestamp}-{lotId}/execution-context.json`

---

### 2. Lot Metadata

Contains auction lot information retrieved from HiBid.

```typescript
interface LotMetadata {
  lotId: string;
  title: string;
  description: string;
  auctionId?: string;
  auctionTitle?: string;
  category?: string;
  images: ImageMetadata[];
  retrievedAt: string;            // ISO 8601 timestamp
  source: "api" | "scrape";       // Data source method
  url: string;                    // Original lot URL
}

interface ImageMetadata {
  imageId: string;                // Unique identifier (hash or sequence)
  url: string;                    // Original image URL
  localPath: string;              // Path relative to execution folder
  size: number;                   // File size in bytes
  width?: number;                 // Image dimensions (if available)
  height?: number;
  format: string;                 // Image format (jpg, png, webp, etc.)
  checksum: string;               // SHA-256 hash for integrity
}
```

**File Location**: `executions/{timestamp}-{lotId}/lot-metadata.json`

---

### 3. Model Configuration

Defines a configured AI model for comparison.

```typescript
interface ModelConfig {
  modelId: string;                // Unique identifier (e.g., "openai-gpt4-vision")
  provider: ModelProvider;
  modelName: string;              // Provider-specific model name
  apiEndpoint?: string;           // Custom endpoint (for SageMaker)
  parameters: ModelParameters;
  enabled: boolean;               // Whether to include in execution
}

enum ModelProvider {
  OPENAI = "openai",
  GOOGLE = "google",
  AWS_SAGEMAKER = "aws_sagemaker"
}

interface ModelParameters {
  temperature?: number;
  maxTokens?: number;
  topP?: number;
  [key: string]: any;             // Provider-specific parameters
}
```

**File Location**: Config file or execution context

---

### 4. Model Request

Represents the request sent to a model.

```typescript
interface ModelRequest {
  modelId: string;
  prompt: string;                 // Rendered prompt template
  images: ImageReference[];
  parameters: ModelParameters;
  timestamp: string;              // ISO 8601
  requestId: string;              // Unique request identifier
}

interface ImageReference {
  imageId: string;
  path: string;                   // Path to local image file
  includeInPrompt: boolean;       // Whether image is included in this request
}
```

**File Location**: `executions/{timestamp}-{lotId}/requests/{modelId}-request.json`

---

### 5. Model Response

Captures the complete response from a model.

```typescript
interface ModelResponse {
  modelId: string;
  requestId: string;
  timestamp: string;              // ISO 8601
  status: ResponseStatus;
  duration: number;               // Response time in milliseconds

  // Response data (if successful)
  content?: string;               // Model's text response
  classification?: Classification;
  metadata?: ResponseMetadata;

  // Error data (if failed)
  error?: ErrorDetails;

  // Performance metrics
  metrics: PerformanceMetrics;
}

enum ResponseStatus {
  SUCCESS = "success",
  ERROR = "error",
  TIMEOUT = "timeout",
  RATE_LIMITED = "rate_limited"
}

interface Classification {
  category: string;
  confidence?: number;
  subcategories?: string[];
  tags?: string[];
  [key: string]: any;             // Model-specific classification data
}

interface ResponseMetadata {
  modelVersion?: string;
  finishReason?: string;
  [key: string]: any;             // Provider-specific metadata
}

interface ErrorDetails {
  code: string;
  message: string;
  retryable: boolean;
  retryAttempts: number;
}
```

**File Location**: `executions/{timestamp}-{lotId}/responses/{modelId}-response.json`

---

### 6. Performance Metrics

Detailed metrics for a model's performance.

```typescript
interface PerformanceMetrics {
  modelId: string;

  // Token usage
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;

  // Cost calculation
  inputCost: number;              // USD
  outputCost: number;             // USD
  totalCost: number;              // USD
  pricingVersion: string;         // Version of pricing table used

  // Timing
  requestStartTime: string;       // ISO 8601
  requestEndTime: string;         // ISO 8601
  duration: number;               // Milliseconds

  // Network
  requestSize: number;            // Bytes
  responseSize: number;           // Bytes
  retryCount: number;
}
```

**File Location**: Embedded in ModelResponse and aggregated in metrics summary

---

### 7. Execution Summary

Aggregated metrics and comparison across all models in an execution.

```typescript
interface ExecutionSummary {
  executionId: string;
  timestamp: string;
  lotId: string;
  modelCount: number;
  successfulModels: number;
  failedModels: number;

  // Aggregated metrics
  totalCost: number;              // USD
  totalTokens: number;
  averageDuration: number;        // Milliseconds
  totalDuration: number;          // Milliseconds (includes parallelization)

  // Per-model summary
  modelSummaries: ModelSummary[];

  // Rankings
  rankings: {
    byCost: string[];             // Model IDs ordered by cost (lowest first)
    bySpeed: string[];            // Model IDs ordered by duration
    byTokenEfficiency: string[];  // Model IDs ordered by tokens used
  };
}

interface ModelSummary {
  modelId: string;
  provider: ModelProvider;
  status: ResponseStatus;
  metrics: PerformanceMetrics;
  classification?: Classification;
}
```

**File Location**: `executions/{timestamp}-{lotId}/execution-summary.json`

---

### 8. Comparison Table

Structured data for the tabular comparison output.

```typescript
interface ComparisonTable {
  executionId: string;
  generatedAt: string;            // ISO 8601

  columns: ColumnDefinition[];
  rows: ComparisonRow[];

  summary: {
    totalCost: number;
    totalTokens: number;
    averageTime: number;
  };
}

interface ColumnDefinition {
  id: string;
  label: string;
  type: "string" | "number" | "currency" | "duration";
  unit?: string;
}

interface ComparisonRow {
  modelId: string;
  modelName: string;
  provider: string;
  status: string;
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  cost: number;                   // USD
  duration: number;               // Milliseconds
}
```

**File Location**: `executions/{timestamp}-{lotId}/comparison-table.json` and `.csv`

---

## Configuration Data Structures

### 9. Application Configuration

System-wide configuration.

```typescript
interface AppConfig {
  version: string;

  // Directories
  executionRootDir: string;       // Root for all execution folders
  cacheDir?: string;              // Optional cache directory

  // API Credentials (references, not actual keys)
  credentials: {
    openai?: CredentialSource;
    google?: CredentialSource;
    aws?: AWSCredentialSource;
    hibid?: CredentialSource;
  };

  // Default behavior
  defaults: {
    parallelExecution: boolean;
    promptTemplate: string;
    retryAttempts: number;
    timeout: number;              // Milliseconds
  };

  // Feature flags
  features: {
    cacheEnabled: boolean;
    batchProcessing: boolean;
    hibidApiEnabled: boolean;
  };

  // Pricing tables (updated periodically)
  pricing: PricingConfig;
}

interface CredentialSource {
  type: "env" | "file" | "aws_secrets" | "gcp_secrets";
  reference: string;              // Env var name, file path, or secret name
}

interface AWSCredentialSource extends CredentialSource {
  region?: string;
  profile?: string;
}

interface PricingConfig {
  version: string;
  lastUpdated: string;            // ISO 8601
  models: {
    [modelId: string]: ModelPricing;
  };
}

interface ModelPricing {
  inputPricePerToken: number;     // USD
  outputPricePerToken: number;    // USD
  currency: string;
}
```

**File Location**: `config/app-config.json` or `.env` file

---

### 10. Profile Configuration

Named configuration profiles for different use cases.

```typescript
interface ProfileConfig {
  profileName: string;
  description: string;

  models: ModelConfig[];
  promptTemplate: string;

  options: {
    includeTitleHint: boolean;
    includeDescriptionHint: boolean;
    maxCost?: number;             // Budget limit in USD
    parallelExecution: boolean;
  };
}
```

**File Location**: `config/profiles/{profile-name}.json`

---

## Data Flow

### Execution Data Flow

1. **Input**: User provides HiBid URL and configuration
2. **Lot Retrieval**: System fetches lot metadata and images → `lot-metadata.json`
3. **Execution Context**: System creates execution context → `execution-context.json`
4. **Model Requests**: System prepares requests for each model → `requests/{modelId}-request.json`
5. **Model Responses**: System captures responses → `responses/{modelId}-response.json`
6. **Metrics Aggregation**: System computes performance metrics → embedded in responses
7. **Execution Summary**: System generates summary → `execution-summary.json`
8. **Comparison Output**: System produces comparison table → `comparison-table.json` and `.csv`

### Directory Structure

```
executions/
└── 2025-11-13-14-30-45-lot123456/
    ├── execution-context.json
    ├── lot-metadata.json
    ├── README.md
    ├── execution.log
    ├── images/
    │   ├── image-001.jpg
    │   ├── image-002.jpg
    │   └── ...
    ├── requests/
    │   ├── openai-gpt4-vision-request.json
    │   ├── google-gemini-pro-request.json
    │   └── aws-sagemaker-request.json
    ├── responses/
    │   ├── openai-gpt4-vision-response.json
    │   ├── google-gemini-pro-response.json
    │   └── aws-sagemaker-response.json
    ├── metrics/
    │   ├── execution-summary.json
    │   ├── comparison-table.json
    │   └── comparison-table.csv
    └── README.md
```

---

## Data Validation

All JSON data structures SHALL be validated against JSON Schema definitions (stored in `schemas/` directory). The application SHALL:

1. Validate all input configuration files on load
2. Validate API responses before persisting
3. Validate execution data before generating summaries
4. Provide clear error messages for validation failures

---

## Data Retention

Default data retention policy:

- Execution folders: Retained indefinitely by default
- Cache data: 7 days (configurable)
- Logs: 30 days (configurable)
- Users can configure auto-cleanup policies in app configuration

---

## Extensibility

The data model is designed for extensibility:

- **Custom Classification Schemas**: Classification interface allows arbitrary properties
- **Provider-Specific Metadata**: ResponseMetadata and ModelParameters support arbitrary keys
- **Future Model Providers**: ModelProvider enum can be extended
- **Additional Metrics**: PerformanceMetrics can be enhanced without breaking changes
