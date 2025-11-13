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

## Practices & Refactoring

This category includes prompts for code review, refactoring, commit message generation, renaming, and shell command suggestions.

### 4. Commit Message Generation (gen-commit-msg.vm)

**Purpose:** Generates concise, clear commit messages following the Conventional Commits specification. Analyzes git diffs and creates structured commit messages with proper scope and type.

**Location:** `/prompts/genius/en/practises/gen-commit-msg.vm`

**Context Variables:**
- `${context.diffContent}` - The git diff to analyze
- `${context.historyExamples}` - User's historical commit message patterns

**Usage:** Automatically generates commit messages when developers stage changes. The AI analyzes the diff content and creates a commit message that follows best practices and the user's historical style.

**Prompt Template:**

```velocity
Given the below code differences (diffs), please generate a concise, clear, and straight-to-the-point commit message.

- Make sure to prioritize the main action.
- Avoid overly verbose descriptions or unnecessary details.
- Start with a short sentence in imperative form, no more than 50 characters long.
- Then leave an empty line and continue with a more detailed explanation, if necessary.

Follow the Conventional Commits specification, examples:

- fix(authentication): fix password regex pattern case
- feat(storage): add support for S3 storage
- test(java): fix test case for user controller
- docs(architecture): add architecture diagram to home page

#if( $context.historyExamples.length > 0 )
Here are the user's historical commit habits:
$context.historyExamples
#end

Diff:

```diff
${context.diffContent}
```
```

---

### 5. Code Review (code-review.vm)

**Purpose:** Provides comprehensive code review feedback focusing on algorithms, logic flow, design decisions, and potential issues. Acts as a senior developer reviewing code changes.

**Location:** `/prompts/genius/en/practises/code-review.vm`

**Context Variables:**
- `${context.frameworkContext}` - Information about frameworks being used
- `${context.stories}` - Related user stories or requirements
- `${context.diffContext}` - The code diff to review

**Usage:** Triggered when developers request code review. Provides critical analysis of code changes, identifies risks, and ensures consistency with existing codebase.

**Prompt Template:**

```velocity
You are a seasoned software developer, and I'm seeking your expertise to review the following code:

- Focus on critical algorithms, logical flow, and design decisions within the code. Discuss how these changes impact the core functionality and the overall structure of the code.
- Identify and highlight any potential issues or risks introduced by these code changes. This will help reviewers pay special attention to areas that may require improvement or further analysis.
- Emphasize the importance of compatibility and consistency with the existing codebase. Ensure that the code adheres to the established standards and practices for code uniformity and long-term maintainability.

${context.frameworkContext}

#if($context.stories.isNotEmpty())
The following user stories are related to these changes:
${context.stories.joinToString("\n")}
#end

${context.diffContext}

As your Tech lead, I am only concerned with key code review issues. Please provide me with a critical summary.
Submit your key insights under 5 sentences in here:
```

---

### 6. Refactoring Suggestions (refactoring.vm)

**Purpose:** Suggests appropriate refactorings to improve code readability, quality, and organization. Uses well-known refactoring patterns from industry best practices.

**Location:** `/prompts/genius/en/practises/refactoring.vm`

**Context Variables:**
- Code snippet passed in context (not explicitly named variable in template)

**Usage:** When developers select code and request refactoring suggestions, this prompt analyzes the code and suggests specific, actionable refactorings with a single consolidated code snippet.

**Prompt Template:**

```velocity
You should suggest appropriate refactorings for the code. Improve code readability, code quality, make the code more organized and understandable.
Answer should contain refactoring description and ONE code snippet with resulting refactoring.
Use well-known refactorings, such as one from this list:
- Renaming
- Change signature, declaration
- Extract or Introduce variable, function, constant, parameter, type parameter
- Extract class, interface, superclass
- Inline class, function, variable, etc
- Move field, function, statements, etc
- Pull up constructor, field, method
- Push down field, method.
Do not generate more than one code snippet, try to incorporate all changes in one code snippet.
Do not generate mock surrounding classes, methods. Do not mock missing dependencies.
Provided code is incorporated into correct and compilable code, don't surround it with additional classes.
Refactor the following code:
```

---

### 7. Rename Suggestions (rename.vm)

**Purpose:** Suggests better, more descriptive names for variables, functions, classes, or other code elements based on their usage and context.

**Location:** `/prompts/genius/en/practises/rename.vm`

**Context Variables:**
- `${context.originName}` - The current name to be replaced
- `${context.language}` - Programming language
- `${context.code}` - Code context where the name is used

**Usage:** Helps developers find more meaningful names for code elements. Returns a single better name suggestion.

**Prompt Template:**

```velocity
${context.originName} is a badname. Please provide 1 better options name for follow code:
```${context.language}
${context.code}
```

just return the better name:
```

---

### 8. Shell Command Suggestions (shell-suggest.vm)

**Purpose:** Converts natural language descriptions into shell commands. Returns only the raw command string ready for execution.

**Location:** `/prompts/genius/en/practises/shell-suggest.vm`

**Context Variables:**
- `${context.today}` - Current date
- `${context.os}` - User's operating system
- `${context.cwd}` - Current working directory
- `${context.shellPath}` - User's shell (bash, zsh, etc.)
- `${context.question}` - User's natural language request

**Usage:** Translates user requests like "undo last git commit" into executable shell commands. Output is designed to be passed directly to subprocess execution.

**Prompt Template:**

```velocity
Return only the command to be executed as a raw string, no string delimiters
wrapping it, no yapping, no markdown, no fenced code blocks, what you return
will be passed to subprocess.check_output() directly.

- Today is: ${context.today}, user system is: ${context.os},
- User current directory is: ${context.cwd}, user use is: ${context.shellPath}, according the tool to create the command.

For example, if the user asks: undo last git commit

You return only line command: git reset --soft HEAD~1

User asks: ${context.question}
```

---

