# Design Document: SupaDev

## Overview

SupaDev is an AI-powered agentic IDE built with SwiftUI and AppKit for macOS, featuring a novel multi-agent pipeline architecture that autonomously researches, designs, codes, verifies, and reviews software projects. The system consists of three primary layers:

1. **IDE Core Layer**: Professional-grade development environment with editor, Git integration, debugger, and terminal
2. **AI Engine Layer**: AI-native features including ghost completions, command bar, error fixing, and smart documentation
3. **Pipeline System Layer**: Six-stage autonomous agent orchestration with specialized AI agents

The architecture emphasizes privacy-first design through a local VaultBot Gateway that routes AI requests while keeping code local. The system uses Git worktrees for safe parallel development, allowing agents to work in isolation while preserving the main development environment.

### Key Design Principles

- **Privacy First**: All code remains local; AI requests are routed through a local gateway
- **Incremental Parsing**: Use Tree-sitter for efficient, incremental syntax analysis
- **Async Everything**: All AI operations and heavy computations run asynchronously to maintain UI responsiveness
- **Agent Specialization**: Each pipeline agent has a specific role and expertise level
- **Safe Isolation**: Git worktrees provide isolated environments for agent experimentation
- **Progressive Enhancement**: Core IDE works without AI; AI features enhance but don't block

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     SupaDev IDE (SwiftUI)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │              │  │              │  │                    │   │
│  │  IDE Core    │  │  AI Engine   │  │  Pipeline System   │   │
│  │              │  │              │  │                    │   │
│  │  - Editor    │  │  - Ghost     │  │  - Research Agent  │   │
│  │  - Git       │  │  - Command   │  │  - Arch Agent      │   │
│  │  - Debugger  │  │  - ErrorFix  │  │  - Coder Agent     │   │
│  │  - Terminal  │  │  - SmartDocs │  │  - Verify Agent    │   │
│  │  - Build     │  │  - Commits   │  │  - QA Agent        │   │
│  │              │  │              │  │  - Orchestrator    │   │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────┘   │
│         │                 │                    │              │
│         └─────────────────┴────────────────────┘              │
│                           │                                    │
│                  ┌────────┴─────────┐                         │
│                  │  WebSocket Layer  │                         │
│                  │  (JSON-RPC)       │                         │
│                  └────────┬─────────┘                         │
└───────────────────────────┼───────────────────────────────────┘
                            │
                   ┌────────┴─────────┐
                   │  VaultBot Gateway │
                   │  (Node.js)        │
                   │                   │
                   │  - Router         │
                   │  - Queue Manager  │
                   │  - Key Manager    │
                   └────────┬─────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────┴────┐      ┌──────┴──────┐    ┌─────┴──────┐
    │ OpenAI  │      │  Anthropic  │    │   Bedrock  │
    │ GPT-4   │      │   Claude    │    │   Models   │
    └─────────┘      └─────────────┘    └────────────┘
```



### Data Flow

1. **User Interaction → IDE Core**: User edits code, triggers builds, manages Git
2. **IDE Core → AI Engine**: Editor context sent for completions, transformations, error fixes
3. **AI Engine → VaultBot Gateway**: AI requests sent via WebSocket JSON-RPC
4. **VaultBot Gateway → AI Providers**: Requests routed to optimal model based on task type
5. **AI Providers → VaultBot Gateway**: Responses returned with generated content
6. **VaultBot Gateway → AI Engine**: Responses delivered back to IDE
7. **AI Engine → IDE Core**: UI updated with completions, diffs, or documentation
8. **Pipeline System → Git Worktree**: Agents commit code to isolated staging branches
9. **Git Worktree → Review Interface**: Changes presented for human approval

## Components and Interfaces

### 1. IDE Core Components

#### 1.1 Editor Engine

**Responsibilities:**
- Syntax highlighting using Tree-sitter incremental parsing
- Text editing operations (insert, delete, undo, redo)
- Cursor management and selection handling
- Line number display and gutter rendering
- Code folding and bracket matching

**Key Interfaces:**

```swift
protocol EditorEngine {
    func loadFile(at path: URL) async throws
    func saveFile() async throws
    func applyEdit(range: Range<Int>, text: String)
    func getSyntaxTree() -> SyntaxTree
    func getVisibleRange() -> Range<Int>
    func highlightRange(range: Range<Int>, style: HighlightStyle)
}

protocol SyntaxHighlighter {
    func parse(source: String, language: Language) -> SyntaxTree
    func updateIncremental(tree: SyntaxTree, edits: [Edit]) -> SyntaxTree
    func getHighlights(tree: SyntaxTree, range: Range<Int>) -> [Highlight]
}

struct SyntaxTree {
    let rootNode: Node
    let language: Language
    let source: String
}

struct Highlight {
    let range: Range<Int>
    let type: HighlightType // keyword, function, variable, string, etc.
}
```

**Implementation Notes:**
- Use Tree-sitter Swift bindings for parsing
- Maintain syntax tree in memory, update incrementally on edits
- Render highlights using SwiftUI AttributedString
- Background thread for parsing, main thread for rendering
- Cache parsed trees per file to avoid re-parsing on file switches

#### 1.2 Git Integration

**Responsibilities:**
- Repository status tracking (modified, staged, untracked files)
- Commit creation and history visualization
- Branch management and worktree operations
- Diff generation and conflict resolution
- Blame annotations and hunk staging

**Key Interfaces:**

```swift
protocol GitService {
    func getStatus() async throws -> RepositoryStatus
    func stageFile(path: String) async throws
    func stageHunk(file: String, hunk: Hunk) async throws
    func commit(message: String) async throws
    func createBranch(name: String) async throws
    func createWorktree(path: String, branch: String) async throws
    func getCommitHistory(limit: Int) async throws -> [Commit]
    func getBlame(file: String) async throws -> [BlameLine]
    func resolveConflict(file: String, resolution: ConflictResolution) async throws
}

struct RepositoryStatus {
    let modified: [String]
    let staged: [String]
    let untracked: [String]
    let conflicted: [String]
}

struct Commit {
    let hash: String
    let author: String
    let date: Date
    let message: String
    let parents: [String]
}

struct Hunk {
    let oldStart: Int
    let oldLines: Int
    let newStart: Int
    let newLines: Int
    let lines: [DiffLine]
}
```

**Implementation Notes:**
- Use libgit2 Swift bindings for Git operations
- Poll repository status every 2 seconds when IDE is active
- Use FileSystemWatcher for immediate change detection
- Render commit graph using custom SwiftUI canvas
- Store worktree metadata in SwiftData for persistence

#### 1.3 Build System

**Responsibilities:**
- Project type detection (Swift, Node.js, Python, etc.)
- Build command execution and output capture
- Error parsing and navigation
- Custom build configuration management

**Key Interfaces:**

```swift
protocol BuildSystem {
    func detectProjectType(at path: URL) async -> ProjectType?
    func build(configuration: BuildConfiguration) async throws -> BuildResult
    func run(configuration: RunConfiguration) async throws -> Process
    func clean() async throws
}

enum ProjectType {
    case swift(package: SwiftPackage)
    case node(packageJson: NodePackage)
    case python(requirements: PythonProject)
    case rust(cargo: CargoProject)
    case go(module: GoModule)
}

struct BuildResult {
    let success: Bool
    let output: String
    let errors: [BuildError]
    let warnings: [BuildWarning]
    let duration: TimeInterval
}

struct BuildError {
    let file: String
    let line: Int
    let column: Int
    let message: String
    let severity: ErrorSeverity
}
```

**Implementation Notes:**
- Use Process API to execute build commands
- Parse compiler output using regex patterns per language
- Stream output in real-time to console view
- Store build configurations in project-local .supadev/build.json

#### 1.4 Debugger

**Responsibilities:**
- LLDB protocol communication
- Breakpoint management
- Variable inspection and expression evaluation
- Call stack navigation
- Step control (over, into, out, continue)

**Key Interfaces:**

```swift
protocol Debugger {
    func attach(to process: Process) async throws
    func setBreakpoint(file: String, line: Int) async throws -> Breakpoint
    func removeBreakpoint(id: String) async throws
    func continue() async throws
    func stepOver() async throws
    func stepInto() async throws
    func stepOut() async throws
    func evaluateExpression(_ expr: String) async throws -> DebugValue
    func getCallStack() async throws -> [StackFrame]
    func getVariables(frame: StackFrame) async throws -> [Variable]
}

struct Breakpoint {
    let id: String
    let file: String
    let line: Int
    let condition: String?
    let enabled: Bool
}

struct StackFrame {
    let index: Int
    let function: String
    let file: String
    let line: Int
}

struct Variable {
    let name: String
    let type: String
    let value: String
    let children: [Variable]?
}
```

**Implementation Notes:**
- Use LLDB Swift bindings or command-line interface
- Maintain breakpoint state in SwiftData
- Update UI reactively when debugger state changes
- Support conditional breakpoints with expression evaluation

#### 1.5 Terminal

**Responsibilities:**
- PTY (pseudo-terminal) process management
- ANSI escape sequence rendering
- Shell integration and command history
- Multiple terminal tab management

**Key Interfaces:**

```swift
protocol Terminal {
    func spawn(shell: String, workingDirectory: URL) async throws -> TerminalSession
    func write(data: Data, to session: TerminalSession) async throws
    func resize(session: TerminalSession, rows: Int, cols: Int) async throws
    func close(session: TerminalSession) async throws
}

protocol TerminalSession {
    var id: UUID { get }
    var outputStream: AsyncStream<Data> { get }
    var isRunning: Bool { get }
}

struct TerminalState {
    var buffer: [[TerminalCell]]
    var cursorRow: Int
    var cursorCol: Int
    var scrollback: [[TerminalCell]]
}

struct TerminalCell {
    let char: Character
    let foreground: Color
    let background: Color
    let bold: Bool
    let italic: Bool
}
```

**Implementation Notes:**
- Use SwiftTerm library for PTY and ANSI handling
- Render terminal using SwiftUI Canvas for performance
- Store terminal history in memory (last 10,000 lines)
- Detect URLs and file paths for clickable links

### 2. AI Engine Components

#### 2.1 Ghost Completions

**Responsibilities:**
- Context extraction from editor
- Completion request generation
- Inline suggestion rendering
- Acceptance and dismissal handling

**Key Interfaces:**

```swift
protocol CompletionProvider {
    func requestCompletion(context: EditorContext) async throws -> Completion?
}

struct EditorContext {
    let filePath: String
    let language: Language
    let cursorPosition: Position
    let textBefore: String // 50 lines before cursor
    let textAfter: String  // 10 lines after cursor
    let recentEdits: [Edit]
}

struct Completion {
    let text: String
    let confidence: Double
    let range: Range<Position>
}

struct Position {
    let line: Int
    let column: Int
}
```

**Implementation Notes:**
- Debounce completion requests (300ms after last keystroke)
- Cancel in-flight requests when user continues typing
- Display completion as gray overlay text at cursor
- Accept on Tab, dismiss on Escape or continued typing
- Cache completions for identical contexts (5-minute TTL)

#### 2.2 Command Bar

**Responsibilities:**
- Natural language instruction parsing
- Code transformation generation
- Diff preview rendering
- Transformation application

**Key Interfaces:**

```swift
protocol CommandBarService {
    func transform(instruction: String, selection: TextSelection) async throws -> Transformation
}

struct TextSelection {
    let range: Range<Position>
    let text: String
    let language: Language
}

struct Transformation {
    let original: String
    let modified: String
    let diff: [DiffHunk]
    let explanation: String
}

struct DiffHunk {
    let type: DiffType // addition, deletion, modification
    let lineRange: Range<Int>
    let content: String
}
```

**Implementation Notes:**
- Show command bar as overlay when ⌘K pressed
- Include surrounding context (100 lines) in transformation request
- Render diff using side-by-side or inline view
- Apply transformation atomically with single undo operation
- Preserve indentation and formatting style

#### 2.3 AI Error Fix

**Responsibilities:**
- Build error parsing and classification
- Fix generation based on error context
- Diff preview and application
- Automatic rebuild triggering

**Key Interfaces:**

```swift
protocol ErrorFixService {
    func analyzeError(_ error: BuildError) async throws -> ErrorAnalysis
    func generateFix(analysis: ErrorAnalysis) async throws -> Fix
}

struct ErrorAnalysis {
    let error: BuildError
    let fixable: Bool
    let confidence: Double
    let context: CodeContext
}

struct Fix {
    let file: String
    let original: String
    let fixed: String
    let explanation: String
}

struct CodeContext {
    let file: String
    let relevantLines: Range<Int>
    let surroundingCode: String
}
```

**Implementation Notes:**
- Parse error messages using language-specific regex patterns
- Extract file path, line number, and error description
- Include 20 lines of context around error location
- Show sparkle icon in build output for fixable errors
- Apply fix and trigger rebuild automatically on acceptance

#### 2.4 Smart Documentation

**Responsibilities:**
- Symbol resolution and context extraction
- Documentation generation
- Popover rendering with formatting

**Key Interfaces:**

```swift
protocol DocumentationService {
    func generateDocs(for symbol: Symbol) async throws -> Documentation
}

struct Symbol {
    let name: String
    let kind: SymbolKind // function, class, method, variable
    let location: Location
    let signature: String?
}

enum SymbolKind {
    case function
    case method
    case `class`
    case `struct`
    case variable
    case constant
}

struct Documentation {
    let summary: String
    let parameters: [Parameter]
    let returnValue: String?
    let examples: [CodeExample]
    let relatedSymbols: [Symbol]
}

struct Parameter {
    let name: String
    let type: String
    let description: String
}
```

**Implementation Notes:**
- Use Tree-sitter to extract symbol definition
- Include implementation code in documentation request
- Render documentation in popover with syntax highlighting
- Cache generated docs per symbol (1-hour TTL)
- Allow user to copy documentation to clipboard

#### 2.5 AI Commit Messages

**Responsibilities:**
- Diff analysis and summarization
- Commit message generation following conventions
- User editing and approval

**Key Interfaces:**

```swift
protocol CommitMessageService {
    func generateMessage(diff: GitDiff) async throws -> CommitMessage
}

struct GitDiff {
    let files: [FileDiff]
    let additions: Int
    let deletions: Int
}

struct FileDiff {
    let path: String
    let status: FileStatus // added, modified, deleted
    let hunks: [Hunk]
}

struct CommitMessage {
    let type: CommitType // feat, fix, refactor, docs, test, chore
    let scope: String?
    let subject: String // max 50 chars
    let body: String?
}

enum CommitType: String {
    case feat, fix, refactor, docs, test, chore, style, perf
}
```

**Implementation Notes:**
- Analyze diff to determine primary change type
- Generate subject line following conventional commits format
- Include detailed body for complex changes (>5 files or >100 lines)
- Show generated message in commit dialog with edit capability
- Learn from user edits to improve future generations

#### 2.6 Intelligent Router

**Responsibilities:**
- Request classification (fast, balanced, powerful)
- Model selection based on task type
- Provider failover and retry logic
- Token usage tracking

**Key Interfaces:**

```swift
protocol IntelligentRouter {
    func route(request: AIRequest) async throws -> AIResponse
    func getOptimalModel(for taskType: TaskType) -> ModelConfig
}

enum TaskType {
    case completion      // fast model required
    case transformation  // balanced model
    case agent          // powerful model
    case documentation  // balanced model
}

struct ModelConfig {
    let provider: AIProvider
    let model: String
    let maxTokens: Int
    let temperature: Double
}

enum AIProvider {
    case openai
    case anthropic
    case google
    case bedrock
    case ollama
}

struct AIRequest {
    let taskType: TaskType
    let prompt: String
    let context: [String: Any]
    let maxTokens: Int?
}

struct AIResponse {
    let content: String
    let model: String
    let tokensUsed: Int
    let latency: TimeInterval
}
```

**Implementation Notes:**
- Route completions to fast models (GPT-3.5, Claude Instant)
- Route transformations to balanced models (GPT-4, Claude Sonnet)
- Route agents to powerful models (GPT-4, Claude Opus)
- Implement exponential backoff for retries (3 attempts max)
- Track token usage per user for quota management
- Fall back to alternative providers on failure

### 3. Pipeline System Components

#### 3.1 Research Agent

**Responsibilities:**
- Task analysis and requirement extraction
- Library and framework research
- Best practice identification
- Research report generation

**Key Interfaces:**

```swift
protocol ResearchAgent {
    func research(task: TaskDescription) async throws -> ResearchReport
}

struct TaskDescription {
    let title: String
    let description: String
    let language: Language?
    let framework: String?
    let constraints: [String]
}

struct ResearchReport {
    let summary: String
    let recommendations: [Recommendation]
    let libraries: [Library]
    let bestPractices: [BestPractice]
    let examples: [CodeExample]
}

struct Recommendation {
    let title: String
    let description: String
    let rationale: String
    let tradeoffs: String
}

struct Library {
    let name: String
    let version: String
    let purpose: String
    let documentation: URL
}
```

**Implementation Notes:**
- Use powerful model (Claude Opus or GPT-4)
- Include web search results if available
- Generate structured report in markdown format
- Store report as artifact in pipeline
- Timeout after 60 seconds



#### 3.2 Architecture Agent

**Responsibilities:**
- Component structure design
- Interface and data model definition
- File structure planning
- Dependency identification

**Key Interfaces:**

```swift
protocol ArchitectureAgent {
    func design(task: TaskDescription, research: ResearchReport) async throws -> ArchitectureDocument
}

struct ArchitectureDocument {
    let overview: String
    let components: [Component]
    let dataModels: [DataModel]
    let fileStructure: FileStructure
    let dependencies: [Dependency]
}

struct Component {
    let name: String
    let responsibility: String
    let interfaces: [Interface]
    let dependencies: [String]
}

struct Interface {
    let name: String
    let methods: [Method]
}

struct Method {
    let signature: String
    let purpose: String
    let parameters: [Parameter]
    let returnType: String
}

struct DataModel {
    let name: String
    let fields: [Field]
    let relationships: [Relationship]
}

struct FileStructure {
    let directories: [String]
    let files: [FileSpec]
}

struct FileSpec {
    let path: String
    let purpose: String
    let components: [String]
}
```

**Implementation Notes:**
- Use powerful model with structured output
- Generate architecture following language conventions
- Include interface definitions in pseudocode
- Create file structure matching project conventions
- Timeout after 45 seconds

#### 3.3 Coder Agent

**Responsibilities:**
- Code implementation from architecture
- Git worktree management
- Incremental commits
- Style consistency

**Key Interfaces:**

```swift
protocol CoderAgent {
    func implement(architecture: ArchitectureDocument, worktree: Worktree) async throws -> ImplementationResult
}

struct Worktree {
    let path: URL
    let branch: String
    let baseBranch: String
}

struct ImplementationResult {
    let filesCreated: [String]
    let filesModified: [String]
    let commits: [Commit]
    let summary: String
}

protocol CodeGenerator {
    func generateFile(spec: FileSpec, architecture: ArchitectureDocument) async throws -> String
    func generateComponent(component: Component, context: CodeContext) async throws -> String
}
```

**Implementation Notes:**
- Create worktree in .supadev/worktrees/{pipeline-id}
- Implement files in dependency order
- Commit after each major component (not per file)
- Use project's existing code style (detect from .editorconfig or existing files)
- Include TODO comments for complex logic requiring human review
- Timeout after 5 minutes (can be extended for large projects)

#### 3.4 Verification Agent

**Responsibilities:**
- Build execution and error detection
- Test suite execution
- Error analysis and fixing
- Verification report generation

**Key Interfaces:**

```swift
protocol VerificationAgent {
    func verify(worktree: Worktree, implementation: ImplementationResult) async throws -> VerificationReport
}

struct VerificationReport {
    let buildSuccess: Bool
    let buildOutput: String
    let buildErrors: [BuildError]
    let testsRun: Int
    let testsPassed: Int
    let testsFailed: Int
    let testOutput: String
    let fixAttempts: [FixAttempt]
    let status: VerificationStatus
}

enum VerificationStatus {
    case passed
    case failedWithFixes
    case failedUnfixable
}

struct FixAttempt {
    let iteration: Int
    let error: BuildError
    let fix: Fix
    let success: Bool
}
```

**Implementation Notes:**
- Check out worktree and run build command
- Parse build errors using BuildSystem component
- Attempt automatic fixes using ErrorFixService
- Maximum 3 fix iterations before escalating
- Run tests if build succeeds
- Generate detailed report with all outputs

#### 3.5 QA Agent

**Responsibilities:**
- Code quality analysis
- Security vulnerability detection
- Best practice verification
- Improvement suggestions

**Key Interfaces:**

```swift
protocol QAAgent {
    func review(worktree: Worktree, verification: VerificationReport) async throws -> QAReport
}

struct QAReport {
    let overallScore: Double // 0.0 to 1.0
    let findings: [Finding]
    let recommendations: [Recommendation]
    let securityIssues: [SecurityIssue]
    let status: QAStatus
}

enum QAStatus {
    case approved
    case approvedWithSuggestions
    case changesRequired
}

struct Finding {
    let severity: Severity
    let category: Category
    let file: String
    let line: Int
    let description: String
    let suggestion: String
}

enum Severity {
    case critical
    case warning
    case suggestion
}

enum Category {
    case codeSmell
    case antiPattern
    case performance
    case security
    case maintainability
}

struct SecurityIssue {
    let type: SecurityIssueType
    let file: String
    let line: Int
    let description: String
    let remediation: String
}

enum SecurityIssueType {
    case sqlInjection
    case xss
    case insecureRandom
    case hardcodedSecret
    case unsafeDeserialization
}
```

**Implementation Notes:**
- Analyze all modified files in worktree
- Check for common security vulnerabilities
- Verify adherence to language-specific best practices
- Generate actionable recommendations
- If critical issues found, send back to Coder Agent with specific instructions
- Maximum 3 re-iteration rounds

#### 3.6 Pipeline Orchestrator

**Responsibilities:**
- Agent sequencing and coordination
- Progress tracking and reporting
- Error handling and recovery
- Artifact management

**Key Interfaces:**

```swift
protocol PipelineOrchestrator {
    func execute(task: TaskDescription) async throws -> PipelineResult
    func pause(pipelineId: UUID) async throws
    func resume(pipelineId: UUID) async throws
    func cancel(pipelineId: UUID) async throws
}

struct PipelineResult {
    let id: UUID
    let status: PipelineStatus
    let stages: [StageResult]
    let worktree: Worktree
    let artifacts: [Artifact]
    let duration: TimeInterval
}

enum PipelineStatus {
    case running
    case paused
    case completed
    case failed
    case cancelled
}

struct StageResult {
    let stage: PipelineStage
    let status: StageStatus
    let startTime: Date
    let endTime: Date?
    let artifact: Artifact?
    let error: Error?
}

enum PipelineStage {
    case research
    case architecture
    case coding
    case verification
    case qa
    case review
}

enum StageStatus {
    case pending
    case running
    case completed
    case failed
}

struct Artifact {
    let stage: PipelineStage
    let type: ArtifactType
    let content: Data
    let metadata: [String: String]
}

enum ArtifactType {
    case researchReport
    case architectureDoc
    case codeCommits
    case verificationReport
    case qaReport
}
```

**Implementation Notes:**
- Execute stages sequentially: Research → Architecture → Coding → Verification → QA
- Store artifacts in .supadev/pipelines/{pipeline-id}/
- Checkpoint after each stage completion
- Support resume from last checkpoint on failure
- Emit progress events for UI updates
- Clean up worktrees after 7 days (configurable)

### 4. VaultBot Gateway

**Responsibilities:**
- WebSocket server for IDE communication
- AI request routing and queuing
- API key management
- Privacy enforcement

**Key Interfaces:**

```typescript
interface VaultBotGateway {
  start(port: number): Promise<void>
  stop(): Promise<void>
  handleRequest(request: AIRequest): Promise<AIResponse>
}

interface RequestQueue {
  enqueue(request: QueuedRequest): void
  dequeue(): QueuedRequest | null
  getPending(): QueuedRequest[]
}

interface QueuedRequest {
  id: string
  request: AIRequest
  priority: Priority
  timestamp: Date
  retries: number
}

enum Priority {
  High = 1,    // Pipeline agents
  Medium = 2,  // Command bar, error fix
  Low = 3      // Completions, documentation
}

interface KeyManager {
  getKey(provider: AIProvider): string | null
  setKey(provider: AIProvider, key: string): void
  validateKey(provider: AIProvider, key: string): Promise<boolean>
}
```

**Implementation Notes:**
- Use ws library for WebSocket server
- Implement JSON-RPC 2.0 protocol
- Queue requests by priority (agents > transformations > completions)
- Store API keys in system keychain (macOS Keychain)
- Never log request content, only metadata
- Support local Ollama for offline mode
- Implement rate limiting per provider

### 5. Data Models

#### 5.1 Project Model

```swift
@Model
class Project {
    @Attribute(.unique) var id: UUID
    var name: String
    var path: URL
    var type: ProjectType
    var lastOpened: Date
    var openFiles: [String]
    var breakpoints: [Breakpoint]
    var buildConfigurations: [BuildConfiguration]
    
    @Relationship(deleteRule: .cascade)
    var pipelines: [Pipeline]
}
```

#### 5.2 Pipeline Model

```swift
@Model
class Pipeline {
    @Attribute(.unique) var id: UUID
    var taskDescription: String
    var status: PipelineStatus
    var currentStage: PipelineStage
    var createdAt: Date
    var completedAt: Date?
    var worktreePath: String
    var artifactsPath: String
    
    @Relationship(inverse: \Project.pipelines)
    var project: Project?
    
    @Relationship(deleteRule: .cascade)
    var stages: [PipelineStageRecord]
}
```

#### 5.3 User Preferences Model

```swift
@Model
class UserPreferences {
    var theme: String
    var fontSize: Int
    var fontFamily: String
    var tabSize: Int
    var insertSpaces: Bool
    var aiProvider: AIProvider
    var aiModel: String
    var telemetryEnabled: Bool
    var autoSave: Bool
    var autoSaveDelay: Int
}
```

#### 5.4 AI Usage Model

```swift
@Model
class AIUsage {
    @Attribute(.unique) var id: UUID
    var date: Date
    var taskType: TaskType
    var provider: AIProvider
    var model: String
    var tokensUsed: Int
    var latency: TimeInterval
    var success: Bool
}
```

## Data Models

### Editor State

The editor maintains state for each open file including cursor position, selection, scroll offset, and undo/redo history. State is persisted to SwiftData on file close and restored on file open.

### Git Repository State

Git state is queried on-demand from libgit2 rather than cached. The only persisted Git data is worktree metadata for pipeline tracking.

### Pipeline Artifacts

Each pipeline stores artifacts in `.supadev/pipelines/{pipeline-id}/`:
- `research.md` - Research report
- `architecture.md` - Architecture document
- `verification.json` - Build and test results
- `qa.json` - Code review findings
- `worktree/` - Git worktree directory

### AI Request Queue

The VaultBot Gateway maintains an in-memory priority queue for AI requests. Requests are not persisted; if the gateway crashes, in-flight requests are lost and must be retried by the IDE.

