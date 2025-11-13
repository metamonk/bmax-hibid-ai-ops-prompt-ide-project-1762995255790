# Functional Requirements

## Priority 0 (P0): Must-Have - MVP Critical

### FR1: HiBid URL Processing
**Description**: The system shall accept and validate HiBid auction lot URLs as input.

**Rationale**: Core functionality required for all use cases. Without URL processing, the tool cannot retrieve data for comparison.

**Acceptance Criteria**:
- Accept URLs via command-line argument (e.g., `npm run compare --url "https://hibid.com/lot/...")`)
- Validate URL format matches HiBid domain pattern
- Extract lot ID from URL structure
- Provide clear error messages for malformed URLs
- Support both HTTP and HTTPS protocols

### FR2: Image and Metadata Retrieval
**Description**: The system shall fetch all images, titles, and descriptions associated with a lot URL.

**Rationale**: Models require image data and contextual information for classification tasks.

**Acceptance Criteria**:
- Retrieve all images associated with the lot (handle multiple images per lot)
- Download images to local execution folder
- Fetch lot title and description text
- Store metadata in structured JSON format
- Implement retry logic for failed downloads (3 retries with exponential backoff)
- Handle missing or incomplete metadata gracefully

### FR3: Multi-Model Execution
**Description**: The system shall support simultaneous execution across multiple AI model providers.

**Rationale**: Core value proposition is comparing models; must support OpenAI, Google Gemini, and AWS SageMaker.

**Acceptance Criteria**:
- Support OpenAI GPT-4 Vision API integration
- Support Google Gemini Pro Vision API integration
- Support AWS SageMaker custom endpoint integration
- Execute API calls in parallel to minimize total runtime
- Configure models via command-line flags or configuration file
- Allow subset model selection (e.g., compare only OpenAI and Gemini)

### FR4: Performance Metrics Collection
**Description**: The system shall track and record token usage, cost, and processing time for each model.

**Rationale**: Essential data for cost optimization and model selection decisions.

**Acceptance Criteria**:
- Track token counts: input tokens, output tokens, total tokens per model
- Calculate costs using current provider pricing tables
- Measure wall-clock time for each model API call
- Record timestamp for each execution
- Handle cases where token counts are not provided by API (estimate if necessary)

### FR5: Tabular Performance Comparison
**Description**: The system shall generate a formatted table comparing performance metrics across all models.

**Rationale**: Primary deliverable for data-driven decision making.

**Acceptance Criteria**:
- Display table in terminal using ASCII table format
- Columns: Model Name, Input Tokens, Output Tokens, Total Tokens, Cost (USD), Time (seconds)
- Include summary row with totals and averages
- Export table to CSV format in execution folder
- Sort models by specified criterion (cost, time, token count)

## Priority 1 (P1): Should-Have - High Value

### FR6: Execution Folder Management
**Description**: The system shall create organized folder structures for capturing execution data.

**Rationale**: Critical for maintaining testing history and enabling reproducibility.

**Acceptance Criteria**:
- Create folder with naming convention: `executions/YYYY-MM-DD-HH-MM-SS-{lotID}/`
- Subfolder structure: `images/`, `responses/`, `metrics/`
- Store downloaded images in `images/` subfolder
- Store model responses in `responses/` subfolder (one JSON file per model)
- Store performance metrics in `metrics/` subfolder
- Generate README.md in execution folder documenting the run

### FR7: API Credential Management
**Description**: The system shall securely manage API credentials for all supported providers.

**Rationale**: Security best practice; prevents credential exposure in command-line arguments.

**Acceptance Criteria**:
- Read credentials from environment variables (OPENAI_API_KEY, GOOGLE_API_KEY, AWS_ACCESS_KEY_ID, etc.)
- Support .env file for local development
- Support configuration file with credential paths (for key files)
- Validate credentials before execution begins
- Never log or display credentials in output
- Provide clear error messages for missing/invalid credentials

### FR8: HiBid API Integration
**Description**: The system shall integrate with HiBid's auction platform API as an alternative to URL scraping.

**Rationale**: More reliable than web scraping; enables production deployment and automation.

**Acceptance Criteria**:
- Authenticate with HiBid API using provided credentials
- Fetch lot data using lot ID
- Retrieve image URLs and metadata via API
- Support both staging and production API endpoints
- Implement rate limiting and backoff strategies
- Fall back to URL scraping if API is unavailable (graceful degradation)

### FR9: Prompt Customization
**Description**: The system shall allow users to customize prompts sent to models with optional parameters.

**Rationale**: Different prompts may be needed for testing; title/description hints can improve accuracy.

**Acceptance Criteria**:
- Accept custom prompt template via command-line argument or file
- Support template variables: {{title}}, {{description}}, {{image_count}}
- Include/exclude title and description as hint in prompts (configurable flag)
- Validate prompt template before execution
- Provide sensible default prompt for image classification

### FR10: Error Handling and Logging
**Description**: The system shall implement comprehensive error handling and logging.

**Rationale**: Critical for debugging, production deployment, and user experience.

**Acceptance Criteria**:
- Log all operations to execution folder (execution.log)
- Use structured logging format (timestamp, level, message, context)
- Distinguish log levels: DEBUG, INFO, WARN, ERROR
- Handle API failures gracefully (don't crash on single model failure)
- Provide user-friendly error messages in terminal
- Include stack traces in log file for debugging
- Exit codes: 0 (success), 1 (user error), 2 (system error)

## Priority 2 (P2): Nice-to-Have - Future Enhancements

### FR11: Batch Processing
**Description**: The system shall support processing multiple lot URLs in a single execution.

**Rationale**: Increases efficiency for testing across multiple examples.

**Acceptance Criteria**:
- Accept multiple URLs via file input (one URL per line)
- Accept multiple URLs via repeated --url flags
- Process lots sequentially with separate execution folders
- Generate aggregate performance report across all lots
- Progress indicator showing batch progress (e.g., "Processing 3 of 10")

### FR12: Response Comparison View
**Description**: The system shall generate a side-by-side comparison of model outputs.

**Rationale**: Helps identify qualitative differences in model classifications.

**Acceptance Criteria**:
- Display model responses in aligned columns in terminal
- Highlight differences in classification results
- Support JSON diff for structured responses
- Export comparison to HTML format for easier review
- Support filtering/searching within responses

### FR13: Configuration Profiles
**Description**: The system shall support named configuration profiles for different use cases.

**Rationale**: Simplifies repeated testing with standard configurations.

**Acceptance Criteria**:
- Define profiles in configuration file (e.g., "fast", "accurate", "cost-optimized")
- Profiles specify: models to use, prompt templates, output options
- Select profile via --profile flag
- List available profiles via --list-profiles command
- Validate profile configuration on load

### FR14: Model Response Caching
**Description**: The system shall optionally cache model responses to avoid redundant API calls.

**Rationale**: Reduces costs and time when re-running analyses with same inputs.

**Acceptance Criteria**:
- Cache responses based on hash of (model, prompt, images)
- Store cache in local directory (e.g., .cache/)
- Provide --no-cache flag to force fresh API calls
- Cache expiration after configurable time period (default 7 days)
- Clear cache via --clear-cache command

### FR15: Cost Budget Enforcement
**Description**: The system shall support setting cost budgets to prevent runaway API expenses.

**Rationale**: Safety feature for production use and cost control.

**Acceptance Criteria**:
- Set per-execution cost limit via --max-cost flag
- Abort execution if estimated cost exceeds limit (before API calls)
- Track cumulative costs across executions (daily/monthly)
- Warn when approaching budget limits
- Support cost budget profiles (dev, staging, prod)

### FR16: Docker Containerization
**Description**: The system shall provide Docker container for deployment flexibility.

**Rationale**: Simplifies deployment, ensures consistent runtime environment.

**Acceptance Criteria**:
- Dockerfile provided in repository
- Container includes all dependencies
- Support environment variable configuration
- Volume mounts for execution folders and config
- Multi-stage build for optimized image size
- Published to container registry (Docker Hub or ECR)
