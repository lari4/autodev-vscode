# AutoDev AI Prompts Documentation

This document provides comprehensive documentation of all AI prompts used in the AutoDev VSCode extension. Prompts are organized by functional categories and include detailed descriptions of their purpose, context variables, and template code.

## Table of Contents

- [Overview](#overview)
- [Prompt Format](#prompt-format)
- [Code Generation](#code-generation)
- [Practices & Refactoring](#practices--refactoring)
- [Error Handling & Debugging](#error-handling--debugging)
- [Search & Retrieval (HyDE)](#search--retrieval-hyde)
- [Page & UI Generation](#page--ui-generation)
- [SQL Generation](#sql-generation)
- [Infrastructure & DevOps](#infrastructure--devops)
- [Model & Reranking](#model--reranking)
- [Quick Actions](#quick-actions)

## Overview

AutoDev uses Apache Velocity Template Language (.vm files) for all AI prompts. The prompts are bilingual (English and Chinese) and support team-level customization through project overrides.

**Total Prompts:** 46 (23 English + 23 Chinese)

**Location:** `/prompts/genius/en/` and `/prompts/genius/zh-cn/`

## Prompt Format

All prompts use the following structure:

```velocity
---
interaction: AppendCursorStream
priority: 100
type: Default
---
```system```
System message here...
```user```
${context.variable}
```

**Supported Interaction Types:**
- `AppendCursor` - Appends result at cursor position
- `AppendCursorStream` - Streams result to cursor position
- `ChatPanel` - Shows result in chat panel
- `OutputFile` - Writes result to file
- `ReplaceSelection` - Replaces selected text

**Supported Chat Roles:** `system`, `user`, `assistant`

---

## Code Generation

This category includes prompts for generating code, documentation, tests, and API data.

### 1. Auto Documentation (auto-doc.vm)

**Purpose:** Generates documentation comments for code based on the provided source code and any existing comments. Supports all programming languages with appropriate comment styles.

**Location:** `/prompts/genius/en/code/auto-doc.vm`

**Context Variables:**
- `${context.language}` - Programming language (e.g., Java, Python, TypeScript)
- `${context.chatContext}` - Additional context from chat history
- `${context.forbiddenRules}` - Rules for what to avoid in documentation
- `${context.startSymbol}` - Comment start symbol (e.g., `/**`, `"""`, `#`)
- `${context.endSymbol}` - Comment end symbol (e.g., `*/`, `"""`)
- `${context.code}` - The source code to document
- `${context.originalComments}` - Existing comments in the code

**Usage:** This prompt is triggered when users request documentation generation for a function, class, or code block. It analyzes the code structure and creates appropriate documentation in the language-specific format.

**Prompt Template:**

```velocity
Write documentation for user's given ${context.language} code.
#if($context.chatContext.length > 0 )
${context.chatContext}
#end
#if($context.forbiddenRules.length > 0)
${context.forbiddenRules}
#end
- Start your documentation with ${context.startSymbol} here, and ends with `${context.endSymbol}`.
Here is User's code:
```${context.language}
${context.code}
```
#if($context.originalComments.length > 0)
Here is code Origin comment: ${context.originalComments}
Please according to the code to update documentation.
#end
Please write documentation for user's code inside the Markdown code block.
```

---

