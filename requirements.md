# CodeSentinel - Requirements Document

## Project Overview

CodeSentinel is an autonomous AI-powered bug testing platform that analyzes Node.js backend applications, sets up testing environments, writes and executes tests, and provides detailed bug reports. Users describe a bug, and the system autonomously validates whether the bug exists in their codebase.

## Core Objectives

1. Automate the bug validation process for Node.js backend applications
2. Reduce manual testing effort by autonomously discovering, setting up, and testing codebases
3. Provide detailed root cause analysis for confirmed bugs
4. Support real-time progress tracking during test execution
5. Enable developers to quickly validate bug reports without manual test writing

## Target Users

- Backend developers working with Node.js applications
- QA engineers validating bug reports
- Development teams needing automated bug verification
- Open source maintainers triaging issue reports

## Functional Requirements

### 1. User Authentication & Repository Management

**FR-1.1**: Users must authenticate via GitHub OAuth using Clerk
- Support for both public and private repositories
- Secure token management for repository access

**FR-1.2**: System must fetch and display user's GitHub repositories
- Show repository metadata (name, owner, language, visibility)
- Filter and search capabilities for repository selection

### 2. Bug Submission & Job Creation

**FR-2.1**: Users must be able to submit bug descriptions with:
- Repository selection from their GitHub account
- Free-form text description of the bug
- Expected vs actual behavior

**FR-2.2**: System must create a job record with:
- Unique job ID
- Initial status (PENDING)
- Timestamp tracking (created, started, completed)
- Association with user and repository

### 3. Autonomous Testing Workflow

**FR-3.1**: Codebase Analysis Phase
- Clone repository into isolated sandbox environment
- Read package.json to identify framework, dependencies, and entry point
- Discover all API endpoints by reading route files
- Identify required environment variables
- Detect database usage (MongoDB, PostgreSQL, etc.)
- Record discovery information for user visibility

**FR-3.2**: Environment Setup Phase
- Install npm dependencies
- Generate test environment variables (JWT secrets, API keys, etc.)
- Provision database instances when needed (MongoDB via createMongoDb tool)
- Create .env file with all required configuration

**FR-3.3**: Server Initialization Phase
- Start application server in background
- Wait for server initialization (8 seconds)
- Obtain public sandbox URL for testing
- Verify server is running and accessible

**FR-3.4**: Test Generation Phase
- Generate one test file per distinct bug in the description
- Use node-fetch for HTTP requests
- Include proper assertions and error handling
- Target actual endpoints discovered during analysis
- Use sandbox URL (not localhost)

**FR-3.5**: Test Execution Phase
- Execute all generated tests sequentially
- Record test results (PASS/FAIL/ERROR)
- Capture exit codes and output
- Continue execution even if tests fail
- Never re-run tests automatically

**FR-3.6**: Bug Analysis Phase
- For failed tests, read source code to identify root cause
- Determine the specific file and function containing the bug
- Assign confidence level (LOW/MEDIUM/HIGH)
- Record detailed bug information

**FR-3.7**: Cleanup & Reporting Phase
- Terminate server processes
- Generate AI summary (2-3 sentences)
- Update job status to COMPLETED or FAILED
- Persist all results to database

### 4. Real-Time Progress Tracking

**FR-4.1**: Job status must update through distinct phases:
- PENDING: Job created, waiting to start
- ANALYZING: Analyzing codebase structure
- SETTING_UP: Installing dependencies and configuring environment
- TESTING: Running tests
- COMPLETED: All tests executed successfully
- FAILED: Job encountered fatal error

**FR-4.2**: UI must poll for updates every 2 seconds during active phases

**FR-4.3**: Display entertaining loading messages during execution

### 5. Results Presentation

**FR-5.1**: Test Results Overview
- Total test count
- Pass/fail statistics
- List of confirmed bugs with severity
- AI-generated summary
- Technical details (framework, entry point, module type, database)

**FR-5.2**: Detailed Bug Reports
- Bug message and description
- Root cause analysis
- Source file location
- Confidence level indicator
- Associated test file

**FR-5.3**: Test Details
- Test file name and test name
- Status badge (PASS/FAIL/ERROR)
- Exit code
- Console output
- Full test code with syntax highlighting
- Download capability for test files

**FR-5.4**: Filtering & Organization
- Tab-based filtering (All/Failed/Passed tests)
- Expandable test cards for detailed view
- Color-coded status indicators

### 6. Data Persistence

**FR-6.1**: Database must store:
- Users (Clerk ID, email, name)
- Repositories (owner, name, URL, metadata)
- Jobs (status, description, summary, discovery info, server info)
- Tests (file, name, content, status, exit code, output)
- Bugs (message, root cause, source file, confidence)
- Usage tracking (points, operations, metadata)

**FR-6.2**: Relationships:
- Users → Repositories (one-to-many)
- Users → Jobs (one-to-many)
- Repositories → Jobs (one-to-many)
- Jobs → Tests (one-to-many)
- Jobs → Bugs (one-to-many)

### 7. Sandbox Environment

**FR-7.1**: E2B sandbox must provide:
- Isolated execution environment per job
- Node.js runtime
- Git for repository cloning
- Network access for package installation
- Configurable timeout (default: appropriate for testing)
- Public URL generation for server access

**FR-7.2**: Sandbox lifecycle:
- Create on job start
- Persist sandbox ID with job
- Automatic cleanup after job completion

### 8. AI Agent Capabilities

**FR-8.1**: Agent must have access to tools:
- terminal: Execute shell commands
- readFiles: Read source code
- createOrUpdateFiles: Create test files
- createEnv: Generate .env files
- createMongoDb: Provision MongoDB instances
- getServerUrl: Obtain public sandbox URL
- updateDiscovery: Save codebase analysis
- updateServerInfo: Track server state
- recordTestResult: Persist test execution results
- recordBug: Record confirmed bugs

**FR-8.2**: Agent must follow structured workflow:
1. Analyze bug report
2. Discover codebase
3. Setup environment
4. Start server
5. Write tests
6. Execute tests
7. Analyze & record bugs
8. Cleanup
9. Write summary

## Non-Functional Requirements

### Performance

**NFR-1**: Job execution should complete within 5-10 minutes for typical repositories

**NFR-2**: UI should remain responsive during polling with 2-second intervals

**NFR-3**: Database queries should be optimized with proper indexing

### Scalability

**NFR-4**: System should support concurrent job execution via Inngest queue

**NFR-5**: Sandbox provisioning should handle multiple simultaneous requests

### Reliability

**NFR-6**: Failed jobs should not crash the system

**NFR-7**: Sandbox timeouts should be handled gracefully

**NFR-8**: Database transactions should ensure data consistency

### Security

**NFR-9**: GitHub tokens must be securely stored and never exposed

**NFR-10**: Sandbox environments must be isolated from each other

**NFR-11**: User data must be protected with proper authentication

**NFR-12**: Environment variables in sandboxes should use test values, never production secrets

### Usability

**NFR-13**: UI should provide clear status indicators at each phase

**NFR-14**: Error messages should be actionable and user-friendly

**NFR-15**: Test results should be easy to understand for non-technical users

**NFR-16**: Code blocks should have syntax highlighting and copy functionality

### Maintainability

**NFR-17**: Code should follow TypeScript best practices

**NFR-18**: Database schema should support migrations

**NFR-19**: Agent prompt should be easily modifiable

**NFR-20**: Tool implementations should be modular and testable

## Technical Constraints

**TC-1**: Must use Next.js 16+ with App Router

**TC-2**: Must use Prisma for database ORM with PostgreSQL

**TC-3**: Must use Inngest for workflow orchestration

**TC-4**: Must use E2B for sandbox environments

**TC-5**: Must use OpenAI GPT-4.1-mini for AI agent

**TC-6**: Must use tRPC for type-safe API

**TC-7**: Must use Clerk for authentication

**TC-8**: Must use Tailwind CSS and shadcn/ui for styling

## Out of Scope

- Frontend application testing
- Non-Node.js applications
- Manual test editing or customization
- Test result comparison across multiple runs
- Integration with CI/CD pipelines
- Automated bug fixing (only detection and reporting)
- Support for languages other than JavaScript/TypeScript
- Real-time collaboration features
- Mobile application support
