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



## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas of redundancy:

1. **Performance properties** (1.2, 5.5, 6.6, 9.5, 21.2) all test timing bounds - these can be consolidated into a general performance property per component
2. **Agent timeout properties** (13.5, 14.7) are specific instances of a general agent execution time property
3. **Iteration limit properties** (17.7, 28.4) both test retry/iteration limits - these follow the same pattern
4. **Round-trip properties** (7.5, 27.2, 28.3) all test state preservation through operations
5. **Worktree isolation properties** (2.9, 15.2) both verify worktree independence

After consolidation, the following properties provide unique validation value:

### Core Editor Properties

**Property 1: Multi-language syntax highlighting**
*For any* supported programming language and valid source code in that language, the IDE_Core should apply syntax highlighting using Tree_Sitter and render it without blocking the UI thread.
**Validates: Requirements 1.1, 1.4**

**Property 2: Language auto-detection**
*For any* file with a recognized extension or shebang, the IDE_Core should automatically detect the correct language and apply appropriate syntax highlighting within 100ms.
**Validates: Requirements 1.2**

**Property 3: Incremental parsing efficiency**
*For any* code edit in a file, the IDE_Core should update syntax highlighting only for the modified region and its dependencies, not the entire file.
**Validates: Requirements 21.6**

**Property 4: Large file responsiveness**
*For any* file up to 10,000 lines, the IDE_Core should render syntax highlighting and respond to keyboard input within 16ms (60 FPS).
**Validates: Requirements 21.1, 21.2**

### Git Integration Properties

**Property 5: File modification indicators**
*For any* file modification (addition, deletion, or change), the IDE_Core should display corresponding gutter marks showing the exact lines affected.
**Validates: Requirements 2.2**

**Property 6: Hunk-level staging independence**
*For any* hunk in a diff, staging that hunk should not affect the staged/unstaged status of other hunks in the same file or other files.
**Validates: Requirements 2.3**

**Property 7: Commit validation**
*For any* commit attempt, if no changes are staged, the IDE_Core should reject the commit with a validation error.
**Validates: Requirements 2.4**

**Property 8: Worktree isolation**
*For any* created worktree, it should have an independent HEAD pointer and working directory such that changes in the worktree do not affect the main working directory until explicitly merged.
**Validates: Requirements 2.8, 2.9, 15.2**

### Build System Properties

**Property 9: Project type detection**
*For any* project directory containing standard configuration files (package.json, Cargo.toml, go.mod, etc.), the IDE_Core should correctly detect the project type.
**Validates: Requirements 3.1**

**Property 10: Build command mapping**
*For any* detected project type, the IDE_Core should execute the correct build command for that type (npm run build, cargo build, go build, etc.).
**Validates: Requirements 3.2**

**Property 11: Build error parsing**
*For any* build failure with compiler output, the IDE_Core should extract file paths, line numbers, and error descriptions to enable navigation.
**Validates: Requirements 3.4, 8.1**

### Debugging Properties

**Property 12: Breakpoint toggle idempotence**
*For any* line in a file, toggling a breakpoint twice should result in the same state as the original (no breakpoint).
**Validates: Requirements 4.2**

**Property 13: Breakpoint pause behavior**
*For any* active breakpoint, when execution reaches that line during debugging, the IDE_Core should pause execution and highlight the current line.
**Validates: Requirements 4.3**

### Terminal Properties

**Property 14: Terminal input forwarding**
*For any* user input in the terminal, the IDE_Core should forward it to the shell process with latency under 10ms.
**Validates: Requirements 5.5**

### Ghost Completion Properties

**Property 15: Completion generation timing**
*For any* typing pause of 300ms or more, the AI_Engine should generate a Ghost_Completion based on surrounding context.
**Validates: Requirements 6.1**

**Property 16: Completion acceptance**
*For any* active Ghost_Completion, pressing Tab should insert the completion text at the cursor position.
**Validates: Requirements 6.3**

**Property 17: Completion dismissal**
*For any* active Ghost_Completion, continuing to type should dismiss the completion without inserting it.
**Validates: Requirements 6.4**

**Property 18: Completion performance**
*For any* 100 completion requests, at least 90 should complete within 500ms.
**Validates: Requirements 6.6**

### Command Bar Properties

**Property 19: Transformation generation**
*For any* natural language instruction and selected code, the AI_Engine should generate a code transformation with a diff preview.
**Validates: Requirements 7.2, 7.3**

**Property 20: Transformation application**
*For any* accepted transformation, the AI_Engine should apply the changes exactly as shown in the diff preview.
**Validates: Requirements 7.4**

**Property 21: Transformation rejection (round-trip)**
*For any* rejected transformation, the editor should restore the exact original code state before the transformation was proposed.
**Validates: Requirements 7.5**

### AI Error Fix Properties

**Property 22: Error fix application and rebuild**
*For any* accepted error fix, the AI_Engine should apply the changes and automatically trigger a rebuild to verify the fix.
**Validates: Requirements 8.5**

### Smart Documentation Properties

**Property 23: Documentation completeness**
*For any* function or method symbol, generated documentation should include purpose, parameters (with types), return value, and at least one usage example.
**Validates: Requirements 9.2**

**Property 24: Documentation performance**
*For any* 100 documentation requests, at least 90 should complete within 1 second.
**Validates: Requirements 9.5**

### AI Commit Message Properties

**Property 25: Conventional commit format**
*For any* diff, the generated commit message should follow the format "type(scope): subject" or "type: subject" where type is one of: feat, fix, refactor, docs, test, chore, style, perf.
**Validates: Requirements 10.2, 10.3**

### Intelligent Router Properties

**Property 26: Request classification determinism**
*For any* AI request of a given type (completion, transformation, agent), the Intelligent_Router should consistently classify it to the same category (fast, balanced, powerful).
**Validates: Requirements 11.1**

**Property 27: Fast model routing**
*For any* Ghost_Completion request, the Intelligent_Router should route it to a fast model with response time under 500ms.
**Validates: Requirements 11.2**

**Property 28: Provider failover**
*For any* AI request that fails due to provider unavailability, the Intelligent_Router should automatically retry with an alternative provider.
**Validates: Requirements 11.6**

### VaultBot Gateway Properties

**Property 29: JSON-RPC protocol compliance**
*For any* message between SupaDev and VaultBot_Gateway, it should conform to the JSON-RPC 2.0 specification with id, method, and params fields.
**Validates: Requirements 12.2**

**Property 30: Privacy invariant - no external storage**
*For any* AI request processed by VaultBot_Gateway, code snippets and responses should never be written to external servers or persistent storage outside the local machine.
**Validates: Requirements 12.5**

### Pipeline Agent Properties

**Property 31: Agent execution time bounds**
*For any* Research_Agent execution, it should complete within 60 seconds, and for any Architecture_Agent execution, it should complete within 45 seconds.
**Validates: Requirements 13.5, 14.7**

**Property 32: Architecture file structure generation**
*For any* architecture design, the Architecture_Agent should generate a file structure specification showing where each component should be implemented.
**Validates: Requirements 14.4**

**Property 33: Coder agent commit creation**
*For any* implementation task, the Coder_Agent should create at least one commit with a descriptive message on the Staging_Branch.
**Validates: Requirements 15.5, 15.7**

**Property 34: Verification build execution**
*For any* worktree with code changes, the Verification_Agent should execute the project's build command and capture the output.
**Validates: Requirements 16.2**

**Property 35: Verification retry limit**
*For any* build error, the Verification_Agent should attempt automatic fixes up to 3 times, and if all attempts fail, escalate to the QA_Agent with error details.
**Validates: Requirements 16.4, 16.7**

**Property 36: QA iteration limit**
*For any* code review with critical issues, the QA_Agent should send code back to the Coder_Agent for fixes up to 3 times, then escalate to human review.
**Validates: Requirements 17.6, 17.7**

### Pipeline Orchestration Properties

**Property 37: Sequential agent execution**
*For any* pipeline, agents should execute in the exact order: Research → Architecture → Coder → Verification → QA, with each stage completing before the next begins.
**Validates: Requirements 18.2**

**Property 38: Artifact persistence**
*For any* completed pipeline stage, artifacts (reports, documents, commits) should be stored in `.supadev/pipelines/{pipeline-id}/` and remain accessible.
**Validates: Requirements 18.4**

**Property 39: Pipeline pause on failure**
*For any* agent failure during pipeline execution, the Pipeline_System should immediately pause and display the error without proceeding to the next stage.
**Validates: Requirements 18.5**

### Review Interface Properties

**Property 40: Approval merge behavior**
*For any* approved pipeline changes, the IDE_Core should merge the Staging_Branch into the main working branch without conflicts (or present conflict resolution if conflicts exist).
**Validates: Requirements 19.4**

**Property 41: Rejection options**
*For any* rejected pipeline changes, the IDE_Core should present exactly three options: discard entirely, send back to specific agent, or manually edit.
**Validates: Requirements 19.5**

### Multi-Language Properties

**Property 42: Language-specific completion style**
*For any* file in a supported language, completions should follow that language's conventions (e.g., camelCase for JavaScript, snake_case for Python).
**Validates: Requirements 20.4**

### Pricing and Quota Properties

**Property 43: Free tier quota enforcement**
*For any* free tier user who exceeds 50 AI queries in a day, the SupaDev should disable AI features until the next reset period (midnight UTC).
**Validates: Requirements 22.6**

### Mobile App Properties

**Property 44: iOS pipeline approval sync**
*For any* pipeline approval or rejection made in the iOS_App, the decision should sync back to the desktop IDE and update the pipeline status.
**Validates: Requirements 23.5**

### Plugin System Properties

**Property 45: Plugin sandboxing**
*For any* plugin attempting unauthorized file system or network access, the IDE_Core should block the access and log a security violation.
**Validates: Requirements 25.5**

**Property 46: Plugin crash isolation**
*For any* plugin crash, the IDE_Core should catch the exception, disable the plugin, and continue operating without affecting other plugins or core functionality.
**Validates: Requirements 25.6**

### Collaboration Properties

**Property 47: Team library accessibility**
*For any* team member with valid credentials, the shared pipeline library should be accessible and display all published pipelines.
**Validates: Requirements 26.2**

### Persistence Properties

**Property 48: Session state round-trip**
*For any* IDE session, saving state on close and restoring on next launch should preserve open files, cursor positions, and window layout exactly.
**Validates: Requirements 27.2**

**Property 49: Pipeline artifact retention**
*For any* pipeline created, artifacts should remain accessible for at least 30 days from creation date.
**Validates: Requirements 27.4**

### Error Handling Properties

**Property 50: Auto-save interval**
*For any* open file with unsaved changes, the IDE_Core should auto-save it every 30 seconds.
**Validates: Requirements 28.2**

**Property 51: Crash recovery round-trip**
*For any* IDE crash with unsaved changes, launching the IDE again should recover and restore all unsaved changes.
**Validates: Requirements 28.3**

**Property 52: Gateway retry with exponential backoff**
*For any* failed AI request, the VaultBot_Gateway should retry up to 3 times with exponentially increasing delays (1s, 2s, 4s).
**Validates: Requirements 28.4**

### Telemetry Properties

**Property 53: Telemetry collection when enabled**
*For any* telemetry-enabled session, the SupaDev should collect anonymous usage statistics (feature usage, error rates, performance metrics).
**Validates: Requirements 30.2**

**Property 54: Telemetry privacy invariant**
*For any* telemetry data collected, it should never contain code content, file names, or project structure information.
**Validates: Requirements 30.3**



## Error Handling

### Error Categories

SupaDev handles errors across multiple layers with different strategies:

#### 1. User Input Errors

**Examples:** Invalid file paths, malformed Git commands, empty commit messages

**Strategy:**
- Validate input before processing
- Display clear, actionable error messages in UI
- Suggest corrections when possible
- Never crash or lose user data

**Implementation:**
```swift
func validateCommitMessage(_ message: String) throws {
    guard !message.trimmingCharacters(in: .whitespaces).isEmpty else {
        throw ValidationError.emptyCommitMessage(
            suggestion: "Please enter a commit message describing your changes"
        )
    }
}
```

#### 2. System Errors

**Examples:** File system permission denied, Git repository not found, process spawn failure

**Strategy:**
- Catch system-level exceptions
- Log detailed error information for debugging
- Display user-friendly message without technical jargon
- Provide recovery actions (e.g., "Check file permissions")

**Implementation:**
```swift
func loadFile(at path: URL) async throws -> String {
    do {
        return try String(contentsOf: path, encoding: .utf8)
    } catch let error as NSError where error.domain == NSCocoaErrorDomain {
        if error.code == NSFileReadNoPermissionError {
            throw FileError.permissionDenied(
                path: path,
                recovery: "Grant read permission to SupaDev in System Settings"
            )
        }
        throw FileError.readFailed(path: path, underlying: error)
    }
}
```

#### 3. AI Service Errors

**Examples:** API rate limit, network timeout, invalid API key, model unavailable

**Strategy:**
- Implement retry logic with exponential backoff
- Fall back to alternative providers when possible
- Cache responses to reduce API calls
- Degrade gracefully (disable AI features but keep IDE functional)

**Implementation:**
```typescript
async function handleAIRequest(request: AIRequest): Promise<AIResponse> {
  let lastError: Error;
  
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      return await routeRequest(request);
    } catch (error) {
      lastError = error;
      
      if (error instanceof RateLimitError) {
        await sleep(Math.pow(2, attempt) * 1000); // Exponential backoff
        continue;
      }
      
      if (error instanceof ProviderUnavailableError) {
        // Try alternative provider
        const fallback = getFallbackProvider(request.provider);
        if (fallback) {
          request.provider = fallback;
          continue;
        }
      }
      
      throw error; // Non-retryable error
    }
  }
  
  throw new MaxRetriesExceededError(lastError);
}
```

#### 4. Pipeline Errors

**Examples:** Agent timeout, build failure, verification failure, QA critical issues

**Strategy:**
- Checkpoint after each stage to enable resume
- Store error context for debugging
- Allow manual intervention and retry
- Escalate to human review after max retries

**Implementation:**
```swift
func executePipeline(task: TaskDescription) async throws -> PipelineResult {
    let pipeline = Pipeline(task: task)
    
    do {
        // Research stage
        let research = try await executeStage(.research) {
            try await researchAgent.research(task: task)
        }
        pipeline.checkpoint(stage: .research, artifact: research)
        
        // Architecture stage
        let architecture = try await executeStage(.architecture) {
            try await architectureAgent.design(task: task, research: research)
        }
        pipeline.checkpoint(stage: .architecture, artifact: architecture)
        
        // Continue with remaining stages...
        
    } catch let error as AgentError {
        pipeline.pause(reason: error.localizedDescription)
        throw PipelineError.agentFailed(stage: pipeline.currentStage, error: error)
    }
    
    return pipeline.result
}

func executeStage<T>(_ stage: PipelineStage, work: () async throws -> T) async throws -> T {
    let timeout: TimeInterval = stage.timeout
    
    return try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask {
            try await work()
        }
        
        group.addTask {
            try await Task.sleep(nanoseconds: UInt64(timeout * 1_000_000_000))
            throw AgentError.timeout(stage: stage, duration: timeout)
        }
        
        guard let result = try await group.next() else {
            throw AgentError.unknown
        }
        
        group.cancelAll()
        return result
    }
}
```

#### 5. Data Corruption Errors

**Examples:** Invalid SwiftData model, corrupted preferences file, malformed pipeline artifact

**Strategy:**
- Validate data on read
- Maintain backup of critical data
- Restore from last known good state
- Notify user of data loss and recovery actions

**Implementation:**
```swift
func loadPreferences() async throws -> UserPreferences {
    do {
        let prefs = try await dataStore.fetch(UserPreferences.self)
        try validatePreferences(prefs)
        return prefs
    } catch {
        logger.error("Failed to load preferences: \(error)")
        
        // Attempt to restore from backup
        if let backup = try? await loadPreferencesBackup() {
            logger.info("Restored preferences from backup")
            return backup
        }
        
        // Fall back to defaults
        logger.warning("Using default preferences")
        return UserPreferences.defaults
    }
}
```

### Error Recovery Strategies

#### Graceful Degradation

When AI services are unavailable, SupaDev continues functioning as a full-featured IDE:
- Editor, Git, debugger, terminal remain fully operational
- AI features show "unavailable" state with clear explanation
- User can configure alternative providers or use offline models

#### State Preservation

All critical state is preserved across errors:
- Auto-save every 30 seconds prevents data loss
- Crash recovery restores unsaved changes
- Pipeline checkpoints enable resume from failure point
- Undo/redo history maintained even after errors

#### User Notification

Errors are communicated clearly without technical jargon:
- Toast notifications for transient errors (e.g., network timeout)
- Modal dialogs for errors requiring user action (e.g., invalid API key)
- Inline error indicators for contextual errors (e.g., build failures)
- Status bar indicators for background errors (e.g., Git fetch failed)

## Testing Strategy

SupaDev employs a comprehensive testing strategy combining unit tests, property-based tests, integration tests, and end-to-end tests.

### Testing Approach

#### Unit Tests

Unit tests verify specific examples, edge cases, and error conditions for individual components.

**Focus Areas:**
- Syntax highlighting for specific language constructs
- Git command parsing and error handling
- Build error message parsing
- JSON-RPC message serialization/deserialization
- Specific UI interactions (button clicks, keyboard shortcuts)

**Example:**
```swift
func testCommitMessageValidation() {
    XCTAssertThrowsError(try validateCommitMessage("")) { error in
        XCTAssertTrue(error is ValidationError)
    }
    
    XCTAssertNoThrow(try validateCommitMessage("feat: add new feature"))
}

func testBuildErrorParsing() {
    let output = "error: file.swift:42:10: expected expression"
    let error = parseBuildError(output)
    
    XCTAssertEqual(error?.file, "file.swift")
    XCTAssertEqual(error?.line, 42)
    XCTAssertEqual(error?.column, 10)
}
```

#### Property-Based Tests

Property-based tests verify universal properties across all inputs using randomized testing.

**Configuration:**
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Use Swift's built-in randomization or QuickCheck-style library

**Focus Areas:**
- Editor operations (insert, delete, undo, redo) maintain consistency
- Git operations preserve repository integrity
- AI transformations preserve code validity
- Pipeline stages execute in correct order
- Worktree isolation prevents cross-contamination

**Example:**
```swift
// Feature: supadev, Property 12: Breakpoint toggle idempotence
func testBreakpointToggleIdempotence() {
    property("Toggling breakpoint twice returns to original state") { (file: String, line: Int) in
        let editor = EditorEngine()
        editor.loadFile(file)
        
        let initialState = editor.hasBreakpoint(at: line)
        
        editor.toggleBreakpoint(at: line)
        editor.toggleBreakpoint(at: line)
        
        let finalState = editor.hasBreakpoint(at: line)
        
        return initialState == finalState
    }.check(iterations: 100)
}

// Feature: supadev, Property 8: Worktree isolation
func testWorktreeIsolation() {
    property("Changes in worktree don't affect main directory") { (changes: [FileChange]) in
        let repo = GitRepository()
        let worktree = repo.createWorktree(branch: "test")
        
        // Apply changes in worktree
        for change in changes {
            worktree.applyChange(change)
        }
        
        // Verify main directory unchanged
        let mainFiles = repo.getModifiedFiles()
        return mainFiles.isEmpty
    }.check(iterations: 100)
}

// Feature: supadev, Property 21: Transformation rejection (round-trip)
func testTransformationRejectionRoundTrip() {
    property("Rejecting transformation restores original code") { (code: String, instruction: String) in
        let editor = EditorEngine()
        editor.setText(code)
        
        let original = editor.getText()
        
        // Generate and reject transformation
        let transformation = await aiEngine.transform(instruction: instruction, selection: editor.selection)
        editor.previewTransformation(transformation)
        editor.rejectTransformation()
        
        let final = editor.getText()
        
        return original == final
    }.check(iterations: 100)
}
```

#### Integration Tests

Integration tests verify interactions between components.

**Focus Areas:**
- Editor ↔ Git integration (gutter marks update on file changes)
- AI Engine ↔ VaultBot Gateway communication
- Pipeline System ↔ Git worktree management
- Build System ↔ Debugger integration

**Example:**
```swift
func testEditorGitIntegration() async throws {
    let project = TestProject()
    let editor = EditorEngine()
    let git = GitService(repository: project.repository)
    
    // Open file and make changes
    try await editor.loadFile(at: project.file("main.swift"))
    editor.insertText("// New comment\n", at: .zero)
    
    // Verify gutter marks appear
    let status = try await git.getStatus()
    XCTAssertTrue(status.modified.contains("main.swift"))
    
    let gutterMarks = editor.getGutterMarks()
    XCTAssertEqual(gutterMarks.count, 1)
    XCTAssertEqual(gutterMarks[0].type, .addition)
}
```

#### End-to-End Tests

End-to-end tests verify complete user workflows.

**Focus Areas:**
- Complete pipeline execution (task → research → architecture → code → review)
- Full editing session (open project → edit → build → debug → commit)
- AI feature workflows (completion → accept, command bar → transform → apply)

**Example:**
```swift
func testCompletePipelineWorkflow() async throws {
    let ide = SupaDevIDE()
    let project = TestProject()
    
    // Initiate pipeline
    let task = TaskDescription(
        title: "Add user authentication",
        description: "Implement JWT-based authentication with login and logout"
    )
    
    let pipeline = try await ide.pipelineSystem.execute(task: task)
    
    // Verify all stages completed
    XCTAssertEqual(pipeline.status, .completed)
    XCTAssertEqual(pipeline.stages.count, 5)
    XCTAssertTrue(pipeline.stages.allSatisfy { $0.status == .completed })
    
    // Verify artifacts exist
    XCTAssertNotNil(pipeline.artifacts.first { $0.type == .researchReport })
    XCTAssertNotNil(pipeline.artifacts.first { $0.type == .architectureDoc })
    XCTAssertNotNil(pipeline.artifacts.first { $0.type == .verificationReport })
    
    // Verify worktree has commits
    let worktree = pipeline.worktree
    let commits = try await ide.git.getCommitHistory(in: worktree, limit: 10)
    XCTAssertGreaterThan(commits.count, 0)
}
```

### Test Organization

```
Tests/
├── UnitTests/
│   ├── EditorTests/
│   │   ├── SyntaxHighlightingTests.swift
│   │   ├── EditOperationsTests.swift
│   │   └── CodeFoldingTests.swift
│   ├── GitTests/
│   │   ├── StatusTrackingTests.swift
│   │   ├── CommitTests.swift
│   │   └── WorktreeTests.swift
│   ├── AIEngineTests/
│   │   ├── CompletionTests.swift
│   │   ├── TransformationTests.swift
│   │   └── ErrorFixTests.swift
│   └── PipelineTests/
│       ├── ResearchAgentTests.swift
│       ├── CoderAgentTests.swift
│       └── OrchestratorTests.swift
├── PropertyTests/
│   ├── EditorPropertyTests.swift
│   ├── GitPropertyTests.swift
│   ├── AIEnginePropertyTests.swift
│   └── PipelinePropertyTests.swift
├── IntegrationTests/
│   ├── EditorGitIntegrationTests.swift
│   ├── AIGatewayIntegrationTests.swift
│   └── PipelineGitIntegrationTests.swift
└── E2ETests/
    ├── PipelineWorkflowTests.swift
    ├── EditingWorkflowTests.swift
    └── AIFeatureWorkflowTests.swift
```

### Continuous Integration

All tests run automatically on every commit:
- Unit tests: ~5 minutes
- Property tests: ~15 minutes (100 iterations each)
- Integration tests: ~10 minutes
- E2E tests: ~20 minutes

Total CI time: ~50 minutes

### Test Coverage Goals

- Unit test coverage: >80% of core logic
- Property test coverage: All 54 correctness properties
- Integration test coverage: All major component interactions
- E2E test coverage: Top 10 user workflows

### Manual Testing

Some aspects require manual testing:
- UI/UX quality and aesthetics
- Performance under real-world load
- Accessibility with screen readers
- Cross-platform compatibility (macOS versions)
- AI response quality and relevance

### Performance Testing

Performance tests verify timing requirements:
- Editor responsiveness (16ms per frame)
- Syntax highlighting speed (100ms for file open)
- AI completion latency (500ms for 90% of requests)
- Build execution time (varies by project)
- Pipeline stage timeouts (60s research, 45s architecture)

**Example:**
```swift
func testCompletionPerformance() async throws {
    let aiEngine = AIEngine()
    var successCount = 0
    
    for _ in 0..<100 {
        let start = Date()
        let completion = try await aiEngine.requestCompletion(context: randomContext())
        let duration = Date().timeIntervalSince(start)
        
        if duration < 0.5 {
            successCount += 1
        }
    }
    
    // 90% should complete within 500ms
    XCTAssertGreaterThanOrEqual(successCount, 90)
}
```

### Security Testing

Security tests verify privacy and sandboxing:
- Plugin sandbox prevents unauthorized file access
- VaultBot Gateway never stores code externally
- Telemetry never includes sensitive data
- API keys stored securely in system keychain

**Example:**
```swift
func testPluginSandboxing() {
    let plugin = TestPlugin()
    let sandbox = PluginSandbox()
    
    // Attempt unauthorized file access
    XCTAssertThrowsError(try sandbox.execute(plugin) {
        try FileManager.default.removeItem(at: URL(fileURLWithPath: "/etc/passwd"))
    }) { error in
        XCTAssertTrue(error is SandboxViolationError)
    }
}

func testTelemetryPrivacy() {
    let telemetry = TelemetryService()
    telemetry.enable()
    
    // Simulate editing session
    let editor = EditorEngine()
    editor.loadFile("MyProject/secret.swift")
    editor.insertText("let apiKey = \"secret123\"")
    
    // Verify telemetry doesn't contain code or filenames
    let data = telemetry.collectData()
    XCTAssertFalse(data.contains("secret.swift"))
    XCTAssertFalse(data.contains("secret123"))
    XCTAssertFalse(data.contains("apiKey"))
}
```

## Deployment and Operations

### Build Configuration

SupaDev uses Xcode for macOS builds:
- Debug: Development builds with logging and debug symbols
- Release: Optimized builds for distribution
- TestFlight: Beta builds with analytics enabled

### Distribution

- Direct download from supadev.dev
- Mac App Store (pending approval)
- Homebrew cask: `brew install --cask supadev`

### VaultBot Gateway Distribution

- npm package: `npm install -g vaultbot-gateway`
- Bundled with IDE (auto-starts on launch)
- Docker image for advanced users

### Monitoring and Analytics

- Crash reporting via Sentry (opt-in)
- Anonymous usage analytics (opt-in)
- Performance metrics (opt-in)
- AI usage tracking for quota management

### Updates

- Automatic update checks on launch
- In-app update notifications
- Staged rollout for major versions
- Rollback capability for failed updates

