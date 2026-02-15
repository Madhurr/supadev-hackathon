# SupaDev — AI-Powered Agentic IDE

> **AI for Bharat Hackathon | Track: AI for Learning & Developer Productivity**

<p align="center">
  <strong>One IDE. Multiple AI Agents. Zero context switching.</strong>
</p>

---

## What is SupaDev?

SupaDev is a native macOS IDE where autonomous AI agents research, architect, code, verify, and review your projects — while you focus on what matters. It's not just autocomplete — it's a team of AI engineers working for you.

## Key Features

### 🤖 Agentic Pipeline System
6-stage autonomous pipeline: Research → Architecture → Code → Verification → QA → Human Review. Each stage runs as an independent AI agent with a specialized role.

### 🧠 AI-Native Editor
- **Ghost Completions** — Tab to accept context-aware suggestions
- **⌘K Command Bar** — Natural language code modification
- **AI Error Fix** — One-click compile error resolution
- **Smart Documentation** — Option+Click AI-generated docs
- **AI Commit Messages** — Meaningful commits from diffs

### 🛠️ Full IDE
- Tree-sitter syntax highlighting for 35+ languages
- Git source control with visual commit graph, blame, worktrees
- LLDB debugger with breakpoints and variable inspection
- Build & Run for Swift, Node, Python, Rust, Go
- 14 premium editor themes

### 🔒 Privacy-First
All code stays local. AI gateway runs on your machine. Your keys, your data.

## Repository Structure

```
├── requirements.md    # Kiro-generated requirements specification
├── design.md          # Kiro-generated technical design document
├── presentation.pdf   # Solution presentation deck
└── README.md          # This file
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| UI | SwiftUI + AppKit (native macOS) |
| Editor | CodeEditSourceEditor + Tree-sitter |
| AI | VaultBot Gateway → Claude/GPT/Gemini/Bedrock |
| Persistence | SwiftData |
| Git | Git CLI integration |
| Debug | LLDB |

## AWS Integration

- **Amazon Bedrock** — Multi-model AI access for agent pipelines
- **AWS Lambda** — Serverless agent execution at scale
- **Amazon S3** — Pipeline artifacts storage
- **AWS CodePipeline** — One-click deployment from IDE

## Impact

- **1.5M+ CS students** in India graduate annually with outdated tools
- **3-5x productivity gain** for routine coding tasks
- **Affordable** alternative to $20/mo AI IDE subscriptions
- **Privacy-first** architecture for enterprise adoption

---

**Built for Bharat's developers. Powered by AI.** ⚡
