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

## 2. HyDE Keywords Search Pipeline

**Purpose:** Semantic code search by extracting and expanding keywords from natural language queries.

**Executor:** `HydeKeywordsStrategy`
**Location:** `/src/code-search/search-strategy/HydeKeywordsStrategy.ts`

**Total Steps:** 3 (2 LLM calls + 1 retrieval)
**Pattern:** Sequential Multi-LLM

### Step-by-Step Flow

```
User Query: "Where is the code for calculating averages?"
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: PROPOSE - Extract and Expand Keywords               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Prompt Template: /prompts/genius/en/hyde/keywords/propose.vm│
│                                                              │
│ Input Context:                                               │
│   {                                                          │
│     question: "Where is the code for calculating averages?" │
│   }                                                          │
│                                                              │
│ LLM processes query step-by-step:                            │
│   1. Resolves pronouns and ambiguous terms                   │
│   2. Generates precise question                              │
│   3. Extracts up to 8 relevant keywords                      │
│   4. Provides variations for each keyword                    │
│                                                              │
│ Output:                                                      │
│   Precise Question:                                          │
│     "Where is calculating the average of a list?"            │
│                                                              │
│   Keywords:                                                  │
│   - basic: ["calculate average", "average", ...]             │
│   - single: ["calculate", "average", "computation"]          │
│   - localization: ["jisuan", "pingjun"]                      │
│                                                              │
│ Output: HydeKeywords object                                  │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: RETRIEVE - Search Using Keywords                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Combined Query Construction:                                 │
│   keywords.basic[0] +                                        │
│   keywords.single[0] +                                       │
│   keywords.localization[0]                                   │
│                                                              │
│ Example: "calculate average calculate jisuan"                │
│                                                              │
│ Retrieval Strategies (parallel):                            │
│   - Full-Text Search                                        │
│   - Semantic/Vector Search                                  │
│   - Git Commit Message Search (if needed)                   │
│                                                              │
│ Retrieved Chunks:                                            │
│   - /src/math/statistics.ts (lines 45-65)                   │
│   - /utils/array-helpers.ts (lines 12-28)                   │
│   - /lib/calculator.ts (lines 89-105)                       │
│                                                              │
│ Output: ChunkItem[]                                          │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: EVALUATE - Answer with Retrieved Code               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Prompt Template: /prompts/genius/en/hyde/keywords/evaluate.vm│
│                                                              │
│ Input Context:                                               │
│   {                                                          │
│     question: "Where is calculating averages?",              │
│     language: "typescript",                                  │
│     code: "<retrieved chunks>",                              │
│     chatContext: "<rules>"                                   │
│   }                                                          │
│                                                              │
│ LLM analyzes code and formulates answer                      │
│                                                              │
│ Output: Answer with file references                          │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
   Display to User
```

### Data Flow Diagram

```
User Query
    │
    ▼
┌────────────────────────┐
│ Prompt: propose.vm     │
│ LLM extracts keywords  │
└────────┬───────────────┘
         │
         ▼
    HydeKeywords {
      question: "...",
      basic: [...],
      single: [...],
      localization: [...]
    }
         │
         ▼
┌────────────────────────┐
│ Combine keywords       │
│ Retrieve code chunks   │
└────────┬───────────────┘
         │
         ▼
    ChunkItem[]
         │
         ▼
┌────────────────────────┐
│ Prompt: evaluate.vm    │
│ LLM synthesizes answer │
└────────┬───────────────┘
         │
         ▼
    Final Answer
```

### Prompts Used

1. **propose.vm** - Keyword extraction and expansion
2. **evaluate.vm** - Answer synthesis from retrieved code

---

## Code Generation Pipelines

These pipelines generate various types of code artifacts using context-augmented single LLM calls.

## 3. Auto Documentation Pipeline

**Purpose:** Generate documentation comments for code elements (functions, classes, methods).

**Executor:** `AutoDocActionExecutor`
**Location:** `/src/action/autodoc/AutoDocActionExecutor.ts`

**Total Steps:** 1 LLM call
**Pattern:** Single + Context Augmentation

### Step-by-Step Flow

```
User Action: Right-click on function → "Generate Documentation"
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: Context Collection                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Parse Document AST                                        │
│    → TreeSitter parser                                       │
│    → Extract NamedElement (function/class)                   │
│                                                              │
│ 2. Extract Code Element                                      │
│    → element.blockRange (function body)                      │
│    → element.identifierRange (signature)                     │
│                                                              │
│ 3. Get Comment Symbols                                       │
│    → Language-specific:                                      │
│      - Java: /** */                                          │
│      - Python: """ """                                       │
│      - JavaScript: /** */                                    │
│                                                              │
│ 4. Collect Toolchain Context                                 │
│    → ToolchainContextManager.collectToolchain()              │
│    → Build tools (package.json, gradle)                      │
│    → Frameworks detected                                     │
│    → Coding conventions                                      │
│                                                              │
│ 5. Build AutoDocTemplateContext                              │
│    {                                                         │
│      language: "typescript",                                 │
│      startSymbol: "/**",                                     │
│      endSymbol: "*/",                                        │
│      code: "<function code>",                                │
│      originalComments: ["..."],                              │
│      chatContext: "<toolchain info>",                        │
│      forbiddenRules: ["no emoji", ...]                       │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: Template Rendering                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ PromptManager.generateInstruction(ActionType.AutoDoc, ctx)   │
│                                                              │
│ Template: /prompts/genius/en/code/auto-doc.vm                │
│                                                              │
│ Velocity rendering:                                          │
│   #if($context.chatContext.length > 0)                       │
│     ${context.chatContext}                                   │
│   #end                                                       │
│                                                              │
│ Rendered Prompt:                                             │
│   "Write documentation for user's given typescript code.     │
│    Start your documentation with /** and ends with */.       │
│    Here is User's code:                                      │
│    ```typescript                                             │
│    function calculateAverage(numbers: number[]): number {    │
│      // code...                                              │
│    }                                                         │
│    ```"                                                      │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: LLM Execution                                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ LanguageModelsService.chat([{                                │
│   role: ChatMessageRole.User,                                │
│   content: renderedPrompt                                    │
│ }])                                                          │
│                                                              │
│ LLM Response:                                                │
│   ```typescript                                              │
│   /**                                                        │
│    * Calculates the arithmetic mean of an array of numbers.  │
│    * @param numbers - Array of numeric values                │
│    * @returns The average of all numbers                     │
│    * @throws Error if array is empty                         │
│    */                                                        │
│   ```                                                        │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 4: Post-Processing                                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Parse Markdown Code Block                                 │
│    → StreamingMarkdownCodeBlock.parse()                      │
│    → Extract content between ``` markers                     │
│                                                              │
│ 2. Format Documentation                                      │
│    → MarkdownTextProcessor.buildDocFromSuggestion()          │
│    → Add proper indentation                                  │
│    → Ensure comment symbols match language                   │
│                                                              │
│ 3. Insert at Proper Location                                 │
│    → insertCodeByRange()                                     │
│    → Position: before function declaration                   │
│                                                              │
│ Final Result:                                                │
│   /**                                                        │
│    * Calculates the arithmetic mean of an array of numbers.  │
│    * @param numbers - Array of numeric values                │
│    * @returns The average of all numbers                     │
│    * @throws Error if array is empty                         │
│    */                                                        │
│   function calculateAverage(numbers: number[]): number {     │
│     // existing code...                                      │
│   }                                                          │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
Named Element (AST Node)
         │
         ▼
┌──────────────────────┐
│ Extract code text    │
│ Get comment symbols  │
│ Collect toolchain    │
└────────┬─────────────┘
         │
         ▼
  AutoDocTemplateContext
         │
         ▼
┌──────────────────────┐
│ Render auto-doc.vm   │
└────────┬─────────────┘
         │
         ▼
     Prompt String
         │
         ▼
┌──────────────────────┐
│ LLM Call             │
└────────┬─────────────┘
         │
         ▼
  Documentation Text
         │
         ▼
┌──────────────────────┐
│ Parse & Format       │
│ Insert before func   │
└──────────────────────┘
```

### Context Variables

- `${context.language}` - Programming language (e.g., "typescript")
- `${context.startSymbol}` - Comment start (e.g., "/**")
- `${context.endSymbol}` - Comment end (e.g., "*/")
- `${context.code}` - The function/class code
- `${context.originalComments}` - Existing comments (if any)
- `${context.chatContext}` - Toolchain context information
- `${context.forbiddenRules}` - Documentation style rules

---

## 4. Auto Test Generation Pipeline

**Purpose:** Generate comprehensive unit tests for classes and methods.

**Executor:** `AutoTestActionExecutor`
**Location:** `/src/action/autotest/AutoTestActionExecutor.ts`

**Total Steps:** 1 LLM call (with streaming)
**Pattern:** Single + Context Augmentation + Streaming

### Step-by-Step Flow

```
User Action: Right-click on class → "Generate Tests"
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: Test Environment Setup                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Detect Test Framework                                     │
│    → TestGenProviderManager.provide(language)                │
│    → Returns language-specific provider:                     │
│      - JavaTestGenProvider (JUnit)                           │
│      - PythonTestGenProvider (pytest)                        │
│      - TypeScriptTestGenProvider (Jest)                      │
│                                                              │
│ 2. Setup Test File                                           │
│    → testgen.setupTestFile(document, element)                │
│    → Determine test file path:                               │
│      src/Calculator.ts → tests/Calculator.test.ts            │
│    → Create file if doesn't exist                            │
│    → Setup imports and boilerplate                           │
│                                                              │
│ 3. Build Initial Context                                     │
│    → targetPath: "tests/Calculator.test.ts"                  │
│    → targetTestClassName: "CalculatorTest"                   │
│    → isNewFile: true/false                                   │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: Context Collection (Multi-Source)                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ A. Source Code Analysis                                      │
│    → TreeSitterFileManager.create(document)                  │
│    → Parse AST to understand structure                       │
│    → Extract imports, dependencies                           │
│                                                              │
│ B. Related Code Discovery                                    │
│    → relevantCodeProviderManager.relatedClassesContext()     │
│    → Find:                                                   │
│      - Classes that current class depends on                 │
│      - Interfaces implemented                                │
│      - Parent classes                                        │
│    → Example: "Calculator depends on Logger, MathUtils"      │
│                                                              │
│ C. Toolchain Context                                         │
│    → ToolchainContextManager.collectToolchain()              │
│    → Collect:                                                │
│      - Test framework configs (jest.config.js)               │
│      - Build tool settings                                   │
│      - Project structure                                     │
│                                                              │
│ D. Language-Specific Test Context                            │
│    → testgen.additionalTestContext()                         │
│    → Provide:                                                │
│      - Test sample code                                      │
│      - Assertion patterns                                    │
│      - Mocking examples                                      │
│                                                              │
│ E. Merge All Contexts                                        │
│    context.chatContext =                                     │
│      toolchainItems.join('\n') +                             │
│      testContextItems.join('\n')                             │
│                                                              │
│ Final AutoTestTemplateContext:                               │
│   {                                                          │
│     language: "typescript",                                  │
│     filename: "Calculator.ts",                               │
│     isNewFile: true,                                         │
│     relatedClasses: "Logger, MathUtils",                     │
│     currentClass: "class Calculator {...}",                  │
│     underTestClassName: "Calculator",                        │
│     targetTestClassName: "CalculatorTest",                   │
│     targetPath: "tests/Calculator.test.ts",                  │
│     imports: ["Logger", "MathUtils"],                        │
│     sourceCode: "class Calculator { add(a,b) {...} }",       │
│     chatContext: "<all collected context>"                   │
│   }                                                          │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: Prompt Generation with Conditionals                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ PromptManager.generateInstruction(ActionType.AutoTest, ctx)  │
│                                                              │
│ Template: /prompts/genius/en/code/test-gen.vm                │
│                                                              │
│ Velocity rendering with conditionals:                        │
│   Write unit test for following typescript code.             │
│   ${context.chatContext}                                     │
│   #if( $context.relatedClasses.length > 0 )                  │
│     Related classes: ${context.relatedClasses}               │
│   #end                                                       │
│   #if( $context.currentClass.length > 0 )                    │
│     Here is current class information:                       │
│     ${context.currentClass}                                  │
│   #end                                                       │
│   Here is the source code to be tested:                      │
│   ```typescript                                              │
│   ${context.sourceCode}                                      │
│   ```                                                        │
│   #if( $context.isNewFile )                                  │
│     Start method test code with typescript Markdown block    │
│   #else                                                      │
│     Start CalculatorTest test code with typescript block     │
│   #end                                                       │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 4: LLM Execution with Streaming                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Open Test File in Editor                                  │
│    → window.showTextDocument(testFilePath)                   │
│    → Navigate before LLM call for real-time updates          │
│                                                              │
│ 2. Start Streaming LLM Call                                  │
│    LanguageModelsService.chat(messages, options, {           │
│      report: async (fragment) => {                           │
│        // Called multiple times as response generates         │
│        output += fragment.part;                              │
│                                                              │
│        // Parse partial code                                 │
│        let parsed = StreamingMarkdownCodeBlock.parse(        │
│          output,                                             │
│          context.language                                    │
│        );                                                    │
│                                                              │
│        if (parsed && parsed.text) {                          │
│          // Update file in real-time!                        │
│          let edit = new WorkspaceEdit();                     │
│          edit.replace(testFile, range, parsed.text);         │
│          workspace.applyEdit(edit);                          │
│        }                                                     │
│      }                                                       │
│    })                                                        │
│                                                              │
│ User sees test code appearing line by line                    │
│                                                              │
│ Final Generated Tests:                                       │
│   describe('Calculator', () => {                             │
│     let calculator: Calculator;                              │
│                                                              │
│     beforeEach(() => {                                       │
│       calculator = new Calculator();                         │
│     });                                                      │
│                                                              │
│     test('should add two numbers correctly', () => {         │
│       expect(calculator.add(2, 3)).toBe(5);                  │
│     });                                                      │
│                                                              │
│     test('should handle negative numbers', () => {           │
│       expect(calculator.add(-2, -3)).toBe(-5);               │
│     });                                                      │
│   });                                                        │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 5: Post-Processing & Validation                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Language-Specific Fixes                                   │
│    → testgen.postProcessCodeFix(testDoc, output)             │
│    → Fix imports                                             │
│    → Add missing assertions                                  │
│    → Format code                                             │
│                                                              │
│ 2. Validation                                                │
│    if (testDoc.getText().length === 0) {                     │
│      // Generation failed - delete empty file                │
│      workspace.fs.delete(testFileUri);                       │
│      throw new Error("Test generation failed");              │
│    }                                                         │
│                                                              │
│ 3. Final State                                               │
│    → Test file exists with complete tests                    │
│    → User can edit and run tests                             │
│    → StatusBar shows "Done"                                  │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
Named Element (Class/Method)
         │
         ▼
┌──────────────────────────────────┐
│ Setup Test Environment           │
│ - Detect framework               │
│ - Create test file path          │
└────────┬─────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│ Collect Multi-Source Context     │
│ - Parse AST                      │
│ - Find related classes           │
│ - Collect toolchain              │
│ - Get test samples               │
└────────┬─────────────────────────┘
         │
         ▼
   AutoTestTemplateContext
         │
         ▼
┌──────────────────────────────────┐
│ Render test-gen.vm               │
│ - Conditional sections           │
└────────┬─────────────────────────┘
         │
         ▼
      Prompt String
         │
         ▼
┌──────────────────────────────────┐
│ LLM Streaming Call               │
│ ┌─────────────────────────────┐  │
│ │ Fragment 1 → Update file    │  │
│ │ Fragment 2 → Update file    │  │
│ │ Fragment 3 → Update file    │  │
│ │ ... (real-time updates)     │  │
│ └─────────────────────────────┘  │
└────────┬─────────────────────────┘
         │
         ▼
   Complete Test Code
         │
         ▼
┌──────────────────────────────────┐
│ Post-Process & Validate          │
│ - Fix imports                    │
│ - Format code                    │
│ - Check not empty                │
└──────────────────────────────────┘
```

### Key Features

**Streaming Updates:**
- User sees tests being generated in real-time
- Each fragment updates the file immediately
- Progressive rendering for better UX

**Context Enrichment:**
- Related classes automatically discovered
- Test framework detected automatically
- Sample test patterns included

**Language Support:**
- Java (JUnit)
- Python (pytest)
- TypeScript/JavaScript (Jest)
- Go (testing package)
- Kotlin, Rust, etc.

---
