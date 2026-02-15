# CodeSentinel - Design Document

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Home Page   │  │ Test Results │  │   Header     │      │
│  │  (Submit)    │  │    Page      │  │  (Auth UI)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                    ┌───────▼────────┐
                    │   tRPC API     │
                    │  (Type-safe)   │
                    └───────┬────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌───────▼────────┐  ┌──────▼──────┐
│  GitHub Router │  │   Jobs Router  │  │ Test Agent  │
│  (Fetch Repos) │  │  (CRUD, Poll)  │  │   Router    │
└───────┬────────┘  └───────┬────────┘  └──────┬──────┘
        │                   │                   │
        │           ┌───────▼────────┐          │
        │           │  Prisma ORM    │          │
        │           │  (PostgreSQL)  │          │
        │           └────────────────┘          │
        │                                       │
        └───────────────────┬───────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Inngest Queue │
                    │  (Orchestrate) │
                    └───────┬────────┘
                            │
                ┌───────────▼───────────┐
                │   Test Agent Function │
                │   (AI Orchestration)  │
                └───────────┬───────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌───────▼────────┐  ┌──────▼──────┐
│  Agent Network │  │   AI Model     │  │  E2B Sandbox│
│  (State Mgmt)  │  │  (GPT-4.1)     │  │  (Isolated) │
└───────┬────────┘  └───────┬────────┘  └──────┬──────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Agent Tools   │
                    │  (10 tools)    │
                    └────────────────┘
```

## Component Design

### 1. Frontend Layer

#### 1.1 Home Page (`src/app/page.tsx`)
**Purpose**: Entry point for bug submission

**Components**:
- Header with authentication status
- Repository selector (dropdown)
- Bug description textarea
- Submit button with loading state
- Feature highlights section

**State Management**:
- `selectedRepo`: Currently selected repository
- `bugDescription`: User's bug description text
- `isLoadingRepos`: Loading state for repository fetch
- `invokeTestAgent`: Mutation state for job creation

**User Flow**:
1. User signs in with GitHub (Clerk)
2. System fetches user's repositories
3. User selects repository from dropdown
4. User enters bug description
5. User clicks "Start Testing"
6. System creates job and redirects to results page

#### 1.2 Test Results Page (`src/app/test/[jobId]/page.tsx`)
**Purpose**: Display real-time progress and final results

**Components**:
- Status badge (PENDING → ANALYZING → SETTING_UP → TESTING → COMPLETED/FAILED)
- Repository information header
- Test results overview (stats cards)
- Bugs summary section
- AI-generated summary
- Technical details (framework, entry point, etc.)
- Detailed bug reports (expandable cards)
- Test list with filtering tabs (All/Failed/Passed)
- Individual test cards with code display

**Polling Strategy**:
- Poll every 2 seconds when status is active (PENDING, ANALYZING, SETTING_UP, TESTING)
- Stop polling when status is terminal (COMPLETED, FAILED)
- Use TanStack Query's `refetchInterval` with conditional logic

**Loading State**:
- Rotating funny messages every 4 seconds
- Smooth fade transitions between messages
- Centered layout with animations

#### 1.3 Header Component (`src/components/header.tsx`)
**Purpose**: Navigation and authentication UI

**Features**:
- Logo/branding
- User authentication status
- Sign in/out buttons (Clerk)
- Navigation links

#### 1.4 Code Block Component (`src/components/code-block.tsx`)
**Purpose**: Syntax-highlighted code display

**Features**:
- Prism.js syntax highlighting
- Copy to clipboard button
- Download button for test files
- Language detection
- Line numbers (optional)

### 2. API Layer (tRPC)

#### 2.1 GitHub Router (`src/trpc/routers/github.ts`)
**Endpoints**:
- `getRepositories`: Fetch user's GitHub repositories
  - Input: None (uses authenticated user)
  - Output: Array of repositories with metadata
  - Uses Octokit to fetch from GitHub API

#### 2.2 Jobs Router (`src/trpc/routers/jobs.ts`)
**Endpoints**:
- `getById`: Fetch job by ID with all related data
  - Input: `{ id: string }`
  - Output: Job with tests, bugs, repository
  - Includes: tests (ordered by createdAt), bugs (ordered by createdAt)

#### 2.3 Test Agent Router (`src/trpc/routers/testAgent.ts`)
**Endpoints**:
- `run`: Create job and trigger Inngest function
  - Input: `{ repoOwner, repoName, repoUrl, bugDescription }`
  - Output: `{ jobId }`
  - Creates Job record with PENDING status
  - Sends event to Inngest queue
  - Returns immediately (async execution)

### 3. Workflow Orchestration (Inngest)

#### 3.1 Test Agent Function (`src/inngest/functions.ts`)
**Event**: `test-agent/run`

**Input**:
```typescript
{
  jobId: string;
  repoUrl: string;
  bugDescription: string;
}
```

**State Schema**:
```typescript
interface TestAgentState {
  jobId: string;
  summary: string;
  testFiles: Record<string, string>;
  discoveryInfo: {
    entryPoint?: string;
    framework?: string;
    moduleType?: string;
    endpoints?: Array<{ method: string; path: string; file: string }>;
    envVarsNeeded?: string[];
    databaseUsed?: boolean;
  };
  serverInfo: {
    port?: number;
    sandboxUrl?: string;
    startCommand?: string;
    isRunning?: boolean;
  };
  testResults: Array<{
    testFile: string;
    testName: string;
    status: 'PASS' | 'FAIL' | 'ERROR';
    exitCode?: number;
    output?: string;
    executedAt?: string;
  }>;
  detectedErrors: Array<{
    testFile: string;
    testName?: string;
    message: string;
    sourceFile?: string;
    rootCause?: string;
  }>;
}
```

**Execution Steps**:
1. Update status to ANALYZING
2. Create E2B sandbox
3. Clone repository
4. Update status to SETTING_UP
5. Initialize agent state
6. Update status to TESTING
7. Run agent network
8. Finalize and save results
9. Update status to COMPLETED/FAILED

**Error Handling**:
- Catch all errors in try-catch
- Update job status to FAILED
- Re-throw error for Inngest retry logic

### 4. AI Agent System

#### 4.1 Agent Configuration
**Model**: OpenAI GPT-4.1-mini
- Temperature: 0.1 (deterministic)
- System prompt: `TEST_AGENT_PROMPT` (detailed workflow instructions)

**Agent Kit Components**:
- `createAgent`: AI agent with tools and lifecycle hooks
- `createState`: Typed state management
- `createNetwork`: Multi-agent orchestration (single agent in this case)

**Lifecycle Hooks**:
- `onResponse`: Extract and save summary when `<task_summary>` tag appears

#### 4.2 Agent Tools

**Terminal Tool** (`src/inngest/tools/terminal.ts`)
- Execute shell commands in sandbox
- Capture stdout and stderr
- Return exit code and output

**Read Files Tool** (`src/inngest/tools/read-files.ts`)
- Read multiple files from sandbox
- Return file contents as array
- Handle file not found errors

**Create/Update Files Tool** (`src/inngest/tools/create-or-update-files.ts`)
- Write files to sandbox filesystem
- Support multiple files in single call
- Create directories as needed

**Create Env Tool** (`src/inngest/tools/create-env.ts`)
- Generate .env file with key-value pairs
- Append to existing .env if present
- Used for non-database environment variables

**Create MongoDB Tool** (`src/inngest/tools/create-mongodb.ts`)
- Provision MongoDB instance in sandbox
- Generate connection URI
- Append to .env with specified variable name
- Return connection details

**Get Server URL Tool** (`src/inngest/tools/get-server-url.ts`)
- Get public URL for sandbox port
- Used for HTTP requests in tests
- Returns `https://{port}-{sandboxId}.e2b.app`

**Update Discovery Tool** (`src/inngest/tools/update-discovery.ts`)
- Save codebase analysis to database
- Update Job.discoveryInfo JSON field
- Called after codebase analysis phase

**Update Server Info Tool** (`src/inngest/tools/update-server-info.ts`)
- Save server state to database
- Update Job.serverInfo JSON field
- Track port, URL, command, running status

**Record Test Result Tool** (`src/inngest/tools/record-test-result.ts`)
- Create Test record in database
- Store test file, name, content, status, exit code, output
- Called after each test execution

**Record Bug Tool** (`src/inngest/tools/record-bug.ts`)
- Create Bug record in database
- Store message, root cause, source file, confidence
- Link to test that discovered the bug
- Called for each confirmed bug

#### 4.3 Agent Prompt Design

**Structure**:
1. Mission statement
2. 9-phase workflow with detailed instructions
3. Tool descriptions
4. Critical rules and constraints
5. Success criteria

**Key Workflow Phases**:
1. Analyze bug report (count issues, identify endpoints)
2. Discover codebase (read package.json, entry file, routes, services)
3. Setup environment (install deps, create .env, provision DB)
4. Start server (background process, wait, get URL)
5. Write tests (one file per bug, use fetch, proper assertions)
6. Execute tests (run once, record all results)
7. Analyze & record bugs (read source, find root cause)
8. Cleanup (kill server, update status)
9. Write summary (2-3 sentences in `<task_summary>` tags)

**Critical Rules**:
- Only provision database if connection code found
- Use exact environment variable names from code
- One test file per distinct bug
- Record every test result (mandatory)
- Record every confirmed bug (mandatory)
- Never re-run tests automatically
- Execute all tests even if some fail

### 5. Database Design

#### 5.1 Schema (Prisma)

**User Model**:
```prisma
model User {
  id      String  @id @default(uuid())
  clerkId String  @unique
  email   String?
  name    String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  repositories Repository[]
  jobs         Job[]
}
```

**Repository Model**:
```prisma
model Repository {
  id     String @id @default(uuid())
  userId String
  
  repoOwner String
  repoName  String
  repoUrl   String
  
  framework  String?
  language   String?
  moduleType String?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  user User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  jobs Job[]
  
  @@unique([userId, repoOwner, repoName])
}
```

**Job Model**:
```prisma
model Job {
  id           String @id @default(uuid())
  userId       String
  repositoryId String
  
  status         JobStatus
  bugDescription String    @db.Text
  summary        String?   @db.Text
  
  discoveryInfo Json?
  serverInfo    Json?
  
  sandboxId  String?
  sandboxUrl String?
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  startedAt   DateTime?
  completedAt DateTime?
  
  user       User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  repository Repository @relation(fields: [repositoryId], references: [id], onDelete: Cascade)
  
  tests Test[]
  bugs  Bug[]
}
```

**Test Model**:
```prisma
model Test {
  id    String @id @default(uuid())
  jobId String
  
  testFile    String
  testName    String
  fileContent String @db.Text
  
  status     TestStatus
  exitCode   Int?
  output     String?    @db.Text
  executedAt DateTime?
  
  createdAt DateTime @default(now())
  
  job Job @relation(fields: [jobId], references: [id], onDelete: Cascade)
}
```

**Bug Model**:
```prisma
model Bug {
  id    String @id @default(uuid())
  jobId String
  
  message    String        @db.Text
  rootCause  String?       @db.Text
  sourceFile String?
  confidence BugConfidence
  
  testFile String?
  testName String?
  
  createdAt DateTime @default(now())
  
  job Job @relation(fields: [jobId], references: [id], onDelete: Cascade)
}
```

**Enums**:
```prisma
enum JobStatus {
  PENDING
  ANALYZING
  SETTING_UP
  TESTING
  COMPLETED
  FAILED
}

enum TestStatus {
  PASS
  FAIL
  ERROR
}

enum BugConfidence {
  LOW
  MEDIUM
  HIGH
}
```

#### 5.2 Indexes
- User: `clerkId` (unique)
- Repository: `userId`, `[userId, repoOwner, repoName]` (unique)
- Job: `userId`, `repositoryId`, `status`, `createdAt`
- Test: `jobId`, `status`
- Bug: `jobId`, `confidence`

#### 5.3 Cascade Deletion
- Delete User → Delete Repositories, Jobs
- Delete Repository → Delete Jobs
- Delete Job → Delete Tests, Bugs

### 6. Sandbox Environment (E2B)

#### 6.1 Template Configuration
**Base**: Custom Dockerfile (`e2b.Dockerfile`)
**Start Command**: `/run.sh` with 30s timeout
**Build Scripts**:
- `build.dev.ts`: Development template build
- `build.prod.ts`: Production template build

#### 6.2 Sandbox Lifecycle
1. Create sandbox with template `code-sentinel-dev`
2. Set timeout (from `SANDBOX_TIMEOUT` constant)
3. Clone repository
4. Execute agent commands
5. Automatic cleanup after job completion

#### 6.3 Sandbox Utilities (`src/inngest/utils.ts`)
- `getSandbox(sandboxId)`: Retrieve existing sandbox instance
- Connection pooling for efficiency
- Error handling for sandbox not found

### 7. Authentication & Authorization

#### 7.1 Clerk Integration
**Provider**: Clerk (`@clerk/nextjs`)
**Strategy**: GitHub OAuth
**Components**:
- `<ClerkProvider>`: Wraps entire app
- `useUser()`: Access current user
- `SignInButton`, `SignOutButton`: Auth UI

#### 7.2 Authorization Flow
1. User signs in with GitHub
2. Clerk creates user session
3. Backend creates User record with `clerkId`
4. GitHub token stored securely by Clerk
5. Token used for repository access via Octokit

#### 7.3 Protected Routes
- All tRPC routes require authentication
- GitHub API calls use user's token
- Jobs are scoped to authenticated user

### 8. UI/UX Design Patterns

#### 8.1 Design System
**Framework**: Tailwind CSS v4
**Component Library**: shadcn/ui (Radix UI primitives)
**Theme**: Custom orange accent color (#f97316)

**Color Palette**:
- Primary: Orange-500 (#f97316)
- Success: Green-500
- Error: Red-500
- Warning: Yellow-500
- Neutral: Gray scale

#### 8.2 Component Patterns
**Cards**: White background, subtle shadow, rounded corners
**Badges**: Color-coded by status/severity
**Buttons**: Primary (orange), secondary (gray), ghost
**Inputs**: Border focus states with orange accent
**Code Blocks**: Dark theme with syntax highlighting

#### 8.3 Responsive Design
- Mobile-first approach
- Breakpoints: sm (640px), md (768px), lg (1024px)
- Grid layouts adapt to screen size
- Touch-friendly button sizes

#### 8.4 Loading States
- Skeleton loaders for initial data fetch
- Spinner for button actions
- Funny rotating messages for long-running jobs
- Progress indicators for multi-phase workflows

#### 8.5 Error Handling
- Toast notifications (Sonner library)
- Inline error messages
- Graceful degradation
- Retry mechanisms

### 9. Data Flow Diagrams

#### 9.1 Job Creation Flow
```
User Input → tRPC Mutation → Create Job (PENDING) → Inngest Event
                                                          ↓
                                                    Queue Job
                                                          ↓
                                                    Execute Agent
                                                          ↓
                                                    Update Status
                                                          ↓
                                                    Save Results
```

#### 9.2 Agent Execution Flow
```
Inngest Trigger → Create Sandbox → Clone Repo → Analyze Code
                                                      ↓
                                                Update Discovery
                                                      ↓
                                            Setup Environment
                                                      ↓
                                                Start Server
                                                      ↓
                                            Update Server Info
                                                      ↓
                                              Generate Tests
                                                      ↓
                                              Execute Tests
                                                      ↓
                                            Record Test Results
                                                      ↓
                                              Analyze Failures
                                                      ↓
                                               Record Bugs
                                                      ↓
                                              Generate Summary
                                                      ↓
                                            Update Job (COMPLETED)
```

#### 9.3 Real-Time Polling Flow
```
User Opens Results Page → Query Job by ID → Check Status
                                                  ↓
                                          Is Active Status?
                                                  ↓
                                    Yes → Poll Every 2s → Update UI
                                                  ↓
                                    No → Stop Polling → Show Final Results
```

## Technology Stack

### Frontend
- **Framework**: Next.js 16.1.6 (App Router)
- **Language**: TypeScript 5
- **UI Library**: React 19.2.3
- **Styling**: Tailwind CSS 4
- **Components**: shadcn/ui (Radix UI)
- **State Management**: TanStack Query 5.90.16
- **Forms**: React Hook Form 7.70.0
- **Validation**: Zod 4.3.5
- **Code Highlighting**: Prism.js 1.30.0
- **Markdown**: markdown-it 14.1.1

### Backend
- **API**: tRPC 11.8.1
- **Database**: PostgreSQL (via Prisma)
- **ORM**: Prisma 6.19.2
- **Authentication**: Clerk 6.37.4
- **Workflow**: Inngest 3.48.1
- **AI**: @inngest/agent-kit 0.13.2
- **Sandbox**: E2B Code Interpreter 2.3.3
- **GitHub API**: Octokit 5.0.5

### Infrastructure
- **Runtime**: Node.js 20+
- **Package Manager**: npm
- **Database**: PostgreSQL with pg adapter
- **Hosting**: Vercel (recommended)
- **Sandbox**: E2B Cloud

## Security Considerations

### 1. Authentication Security
- GitHub OAuth via Clerk (industry-standard)
- Secure token storage (encrypted by Clerk)
- Session management with automatic expiration
- CSRF protection built into Next.js

### 2. Sandbox Isolation
- Each job runs in isolated E2B sandbox
- No shared state between sandboxes
- Automatic cleanup after execution
- Network isolation from production systems

### 3. Environment Variables
- Test values only (never production secrets)
- Random generation for JWT secrets, salts
- Database instances are ephemeral
- No persistence of sensitive data

### 4. Data Access Control
- Jobs scoped to authenticated user
- Repository access validated via GitHub token
- Database queries filtered by user ID
- Cascade deletion prevents orphaned data

### 5. Input Validation
- Zod schemas for all tRPC inputs
- Prisma type safety for database operations
- Sanitization of user-provided bug descriptions
- File path validation in sandbox operations

### 6. Rate Limiting
- Inngest queue prevents overwhelming system
- GitHub API respects rate limits
- E2B sandbox quotas enforced
- Database connection pooling

## Performance Optimization

### 1. Database
- Indexed queries on frequently accessed fields
- Eager loading with Prisma includes
- Connection pooling via pg adapter
- JSON fields for flexible schema

### 2. Frontend
- Server-side rendering for initial load
- Client-side navigation with Next.js
- Code splitting by route
- Image optimization (Next.js built-in)

### 3. API
- tRPC batching for multiple queries
- TanStack Query caching
- Conditional polling (only when active)
- Optimistic updates where applicable

### 4. Sandbox
- Reuse sandbox instances when possible
- Parallel tool execution where safe
- Efficient file operations (batch reads/writes)
- Timeout management

## Monitoring & Observability

### 1. Logging
- Inngest execution logs
- Agent tool invocations
- Database query logs
- Error stack traces

### 2. Metrics
- Job completion rates
- Average execution time
- Test pass/fail ratios
- Sandbox resource usage

### 3. Error Tracking
- Failed job analysis
- Sandbox timeout tracking
- API error rates
- Database connection issues

## Future Enhancements

### 1. Multi-Language Support
- Python backend testing
- Java/Spring applications
- Ruby on Rails
- Go services

### 2. Advanced Testing
- Integration test generation
- Load testing capabilities
- Security vulnerability scanning
- Performance regression detection

### 3. Collaboration Features
- Team workspaces
- Shared test results
- Comments and annotations
- Test result comparison

### 4. CI/CD Integration
- GitHub Actions integration
- GitLab CI support
- Webhook notifications
- Automated PR comments

### 5. Enhanced Reporting
- PDF export
- Custom report templates
- Historical trend analysis
- Slack/Discord notifications
