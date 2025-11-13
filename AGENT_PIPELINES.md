# AutoDev Agent Pipelines Documentation

This document provides comprehensive documentation of all agent pipelines and workflows in the AutoDev VSCode extension. It describes how prompts are chained together, what data flows between them, and how the AI agent processes different types of user requests.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline Patterns](#pipeline-patterns)
- [Search Pipelines](#search-pipelines)
  - [HyDE Code Search Pipeline](#1-hyde-code-search-pipeline)
  - [HyDE Keywords Search Pipeline](#2-hyde-keywords-search-pipeline)
- [Code Generation Pipelines](#code-generation-pipelines)
- [Refactoring Pipelines](#refactoring-pipelines)
- [Custom Action Pipeline](#custom-action-pipeline)
- [Common Patterns](#common-patterns)
- [Data Flow Diagrams](#data-flow-diagrams)

## Overview

AutoDev uses multiple AI agent pipelines to handle different types of developer tasks. These pipelines range from simple single-LLM calls to complex multi-step workflows involving multiple AI invocations, code retrieval, and context augmentation.

**Key Statistics:**
- **Total Pipelines:** 10 major pipelines
- **Pipeline Types:** Sequential, Single + Context, Iterative
- **Average Steps:** 1-3 LLM calls per pipeline
- **Streaming Support:** Yes (for test generation and chat)

## Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    AutoDev Extension                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐      ┌─────────────────┐               │
│  │ User Interface │──────│  Command Service│               │
│  └────────────────┘      └─────────────────┘               │
│          │                        │                         │
│          ▼                        ▼                         │
│  ┌────────────────┐      ┌─────────────────┐               │
│  │ Chat Interface │      │ Action Executors│               │
│  │   (Catalyser)  │      │  (AutoDoc, etc) │               │
│  └────────────────┘      └─────────────────┘               │
│          │                        │                         │
│          └────────┬───────────────┘                         │
│                   ▼                                          │
│          ┌─────────────────┐                                │
│          │ Prompt Manager  │                                │
│          │ - Template Load │                                │
│          │ - Context Build │                                │
│          │ - Rendering     │                                │
│          └─────────────────┘                                │
│                   │                                          │
│                   ▼                                          │
│          ┌─────────────────┐                                │
│          │ Language Models │                                │
│          │    Service      │                                │
│          │  (LLM Gateway)  │                                │
│          └─────────────────┘                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pipeline Execution Flow

```
User Action/Command
       │
       ▼
┌──────────────┐
│Context       │ ← Collect document, selection, AST
│Collection    │
└──────────────┘
       │
       ▼
┌──────────────┐
│Toolchain     │ ← Build configs, frameworks, conventions
│Context       │
└──────────────┘
       │
       ▼
┌──────────────┐
│Template      │ ← Load .vm file, render with Velocity
│Rendering     │
└──────────────┘
       │
       ▼
┌──────────────┐
│LLM           │ ← Call AI model (streaming or single)
│Execution     │
└──────────────┘
       │
       ▼
┌──────────────┐
│Post-         │ ← Parse, format, validate
│Processing    │
└──────────────┘
       │
       ▼
┌──────────────┐
│UI Update     │ ← Insert code, show results
└──────────────┘
```

## Pipeline Patterns

AutoDev implements three main pipeline patterns:

### 1. Sequential Multi-LLM Pattern

**Characteristics:**
- Multiple LLM calls in sequence
- Output of one step becomes input to next
- Used for complex reasoning tasks

**Example:** HyDE Search (Propose → Retrieve → Evaluate)

### 2. Single + Context Augmentation Pattern

**Characteristics:**
- Single LLM call with enriched context
- Extensive pre-processing to gather context
- Most common pattern in AutoDev

**Example:** Auto Documentation, Auto Test

### 3. Iterative Pattern

**Characteristics:**
- Same LLM call repeated for multiple items
- Parallel or sequential execution
- Aggregates results

**Example:** LLM Reranking

---

## Search Pipelines

These pipelines implement semantic code search using the HyDE (Hypothetical Document Embeddings) technique.

## 1. HyDE Code Search Pipeline

**Purpose:** Semantic code search by generating hypothetical code that matches the query, then finding similar real code.

**Executor:** `HydeCodeStrategy`
**Location:** `/src/code-search/search-strategy/HydeCodeStrategy.ts`

**Total Steps:** 3 (2 LLM calls + 1 retrieval)
**Pattern:** Sequential Multi-LLM

### Step-by-Step Flow

```
User Query: "How do we handle authentication?"
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: PROPOSE - Generate Hypothetical Code                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Prompt Template: /prompts/genius/en/hyde/code/propose.vm    │
│                                                              │
│ Input Context:                                               │
│   {                                                          │
│     question: "How do we handle authentication?",            │
│     chatContext: "<current file context>"                    │
│   }                                                          │
│                                                              │
│ LLM generates hypothetical code (5-10 lines):                │
│   ```javascript                                              │
│   const auth = require('./auth');                            │
│   app.use(auth.middleware);                                  │
│   app.post('/login', auth.verify);                           │
│   ```                                                        │
│                                                              │
│ Output: HydeDocument { content: "...", language: "js" }      │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: RETRIEVE - Find Similar Code                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Retrieval Strategies (parallel):                            │
│                                                              │
│ ┌──────────────────┐  ┌──────────────────┐                 │
│ │ Full-Text Search │  │ Semantic Search  │                 │
│ │  (FTS Engine)    │  │ (Vector DB)      │                 │
│ └──────────────────┘  └──────────────────┘                 │
│         │                      │                            │
│         └──────────┬───────────┘                            │
│                    ▼                                         │
│         ┌──────────────────┐                                │
│         │ Deduplication    │                                │
│         │ & Ranking        │                                │
│         └──────────────────┘                                │
│                    │                                         │
│ Retrieved Chunks:                                            │
│   - /src/auth/middleware.ts (lines 10-30)                   │
│   - /src/auth/verify.ts (lines 5-25)                        │
│   - /config/auth.config.ts (lines 1-15)                     │
│                                                              │
│ Output: ChunkItem[] (max 10 chunks)                         │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: EVALUATE - Answer Query with Retrieved Code         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Prompt Template: /prompts/genius/en/hyde/code/evaluate.vm   │
│                                                              │
│ Input Context:                                               │
│   {                                                          │
│     question: "How do we handle authentication?",            │
│     language: "typescript",                                  │
│     code: "<concatenated chunks>",                           │
│     chatContext: "<formatting rules>"                        │
│   }                                                          │
│                                                              │
│ LLM analyzes and synthesizes answer:                         │
│   "Authentication is handled through middleware in           │
│    /src/auth/middleware.ts. The verify() function validates  │
│    tokens using JWT. Here's the key code:                    │
│    [code snippet from chunks]"                               │
│                                                              │
│ Output: Final answer with code references                    │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
   Display to User
```

### Data Flow Diagram

```
┌──────────────┐
│ User Query   │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  Context: { question, chatContext }                  │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  Prompt 1: propose.vm                                │
│  → LLM Call #1                                       │
│  → Output: Hypothetical Code String                  │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  Retrieval: DefaultRetrieval.retrieve()              │
│  → Input: hypothetical code                          │
│  → FTS Search                                        │
│  → Vector Search                                     │
│  → Output: ChunkItem[]                               │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  Context: { question, code: chunks, chatContext }    │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  Prompt 2: evaluate.vm                               │
│  → LLM Call #2                                       │
│  → Output: Final Answer                              │
└──────┬───────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│ Show Answer  │
└──────────────┘
```

### Context Variables

**Propose Phase:**
- `${context.question}` - User's search query

**Evaluate Phase:**
- `${context.question}` - Original query
- `${context.language}` - Programming language
- `${context.code}` - Retrieved code chunks (concatenated)
- `${context.chatContext}` - Formatting rules and guidelines

### Prompts Used

1. **propose.vm** - Generates hypothetical code snippet
   - Role: Creative code synthesis
   - Output: 5-10 lines of code

2. **evaluate.vm** - Analyzes retrieved code and answers query
   - Role: Code comprehension and summarization
   - Output: Natural language answer with code references

### Key Implementation Details

**File:** `/src/code-search/search-strategy/HydeCodeStrategy.ts:31-70`

```typescript
async search(query: string, context: SearchContext): Promise<StrategyFinalPrompt> {
  // Step 1: Propose
  const instruction = await renderHydeTemplate(
    HydeStep.Propose,
    SystemActionType.SemanticSearchCode,
    { question: query, chatContext: context.currentDoc }
  );
  const hydeDoc = await executeIns(this.ext, instruction);

  // Step 2: Retrieve
  const chunks = await retrieveChunks(
    hydeDoc.content,
    this.retrievalSettings
  );

  // Step 3: Evaluate
  const evalInstruction = await renderHydeTemplate(
    HydeStep.Evaluate,
    SystemActionType.SemanticSearchCode,
    {
      question: query,
      code: chunks.map(c => c.text).join('\n'),
      language: context.language,
      chatContext: context.currentDoc
    }
  );

  return new StrategyFinalPrompt(evalInstruction, chunks);
}
```

### Performance Characteristics

- **Latency:** 4-8 seconds (2 LLM calls + retrieval)
- **Accuracy:** High (better than keyword search)
- **Cost:** 2x LLM calls compared to single-shot search
- **Cache:** Retrieval results can be cached

### Use Cases

- "Where is the authentication logic?"
- "How do we connect to the database?"
- "Show me error handling code"
- "Find the API endpoint for user creation"

---

