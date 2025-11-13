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

### 2. API Test Data Generation (gen-api-data.vm)

**Purpose:** Generates JSON test data for API endpoints based on code analysis. Creates request/response examples that can be used for API testing and documentation.

**Location:** `/prompts/genius/en/code/gen-api-data.vm`

**Context Variables:**
- `${context.language}` - Programming language
- `${context.baseUrl}` - Base URL for the API route
- `${context.requestStructure}` - Information about request body structure
- `${context.responseStructure}` - Information about response body structure
- `${context.selectedText}` - The selected API code

**Usage:** Used when developers need to generate test data for API endpoints. The prompt analyzes the endpoint code and creates realistic JSON request/response examples.

**Prompt Template:**

```velocity
Generate JSON data (with markdown code block) based on given ${context.language} code and request/response info.
So that we can use it to test for APIs.

Use the following template to generate the JSON data:

action: // request method, url: // the request url
request:
    // the request body in json
response:
    // the response body in json

For example:

```
GET /api/v1/users
request:
{
  "body": "hello"
}
```
Here is code information:
#if($context.baseUrl.length > 0)
// base URL route:
#end
// compare this request body relate info: ${context.requestStructure}
// compare this response body relate info: ${context.responseStructure}
Here is the user's code:
```${context.language}
${context.selectedText}
```

Please generate the JSON data.
```

---

### 3. Unit Test Generation (test-gen.vm)

**Purpose:** Generates comprehensive unit tests for given source code. Analyzes the code structure, dependencies, and creates appropriate test cases using the project's testing framework.

**Location:** `/prompts/genius/en/code/test-gen.vm`

**Context Variables:**
- `${context.language}` - Programming language
- `${context.chatContext}` - Additional context and requirements
- `${context.relatedClasses}` - Information about related classes/dependencies
- `${context.currentClass}` - Current class being tested
- `${context.imports}` - Libraries and frameworks used in the project
- `${context.sourceCode}` - The source code to test
- `${context.isNewFile}` - Whether to create a new test file
- `${context.underTestClassName}` - Name of the class being tested

**Usage:** Triggered when developers want to automatically generate unit tests. The prompt considers existing test patterns, dependencies, and creates tests that follow the project's conventions.

**Prompt Template:**

```velocity
Write unit test for following ${context.language} code.
${context.chatContext}
#if( $context.relatedClasses.length > 0 )
${context.relatedClasses}
#end
#if( $context.currentClass.length > 0 )
Here is current class information:
${context.currentClass}
#end
// here is the user used libraries
// ${context.imports}

// Here is the source code to be tested:
```$context.language
${context.sourceCode}
```

## if newFile
#if( $context.isNewFile )
Start method test code with ${context.language} Markdown code block here:
#else
Start ${context.underTestClassName} test code with ${context.language} Markdown code block here:
#end
```

---

