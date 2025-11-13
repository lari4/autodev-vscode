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

## Error Handling & Debugging

This category contains prompts for debugging runtime errors and fixing code issues.

### 9. Fix Runtime Errors (fix-error.vm)

**Purpose:** Analyzes console logs and error messages to identify root causes of runtime problems and provides long-term solutions. Acts as an expert debugging assistant.

**Location:** `/prompts/genius/en/error/fix-error.vm`

**Context Variables:**
- `${context.errorText}` - Console output/error messages
- `${context.soureCode}` - The program code causing the error

**Usage:** Triggered when runtime errors occur. Analyzes error logs and code to provide fixes. Prefers code modifications over suggesting library installations.

**Prompt Template:**

```velocity
As a helpful assistant with expertise in code debugging, your objective is to identify the roots of runtime problems by analyzing console logs and providing general solutions to fix the issues. When assisting users, follow these rules:

1. Always be helpful and professional.
2. Use your mastery in code debugging to determine the cause of runtime problems by looking at console logs.
3. Provide fixes to the bugs causing the runtime problems when given the code.
4. Ensure that your solutions are not temporary "duct tape" fixes, but instead, provide long-term solutions.
5. If a user sends you a one-file program, append the fixed code in markdown format at the end of your response.
This code will be extracted using re.findall(r"`{{3}}(\w*)\n([\S\s]+?)\n`{{3}}", model_response)
so adhere to this formatting strictly.
6. If you can fix the problem strictly by modifying the code, do so. For instance, if a library is missing, it is preferable to rewrite the code without the library rather than suggesting to install the library.
7. Always follow these rules to ensure the best assistance possible for the user.

Now, consider this user request:

"Please help me understand what the problem is and try to fix the code. Here's the console output and the program text:

Console output:
%s
Texts of programs:
%s
Provide a helpful response that addresses the user's concerns, adheres to the rules, and offers a solution for the runtime problem.

```
${context.errorText}
```

```
${context.soureCode}
```
```

---

## Search & Retrieval (HyDE)

This category uses Hypothetical Document Embeddings (HyDE) technique for semantic code search. It includes prompts for both code and keyword-based search with proposal and evaluation phases.

### 10. Code Search Proposal (hyde/code/propose.vm)

**Purpose:** Generates hypothetical code snippets that could answer a code search query. Used in the first phase of HyDE search to create embedding targets.

**Location:** `/prompts/genius/en/hyde/code/propose.vm`

**Context Variables:**
- `${context.question}` - The user's search query

**Usage:** Part of HyDE (Hypothetical Document Embeddings) search pipeline. Generates synthetic code examples that semantically match the query for better vector search.

**Prompt Template:**

```velocity
Write a code snippet that could hypothetically be returned by a code search engine as the answer to the query: {query}

- Write the snippets in a programming or markup language that is likely given the query
- The snippet should be between 5 and 10 lines long
- Surround the snippet in triple backticks

**Examples**
User: What's the Qdrant threshold?
Response:
```rust
SearchPoints {
    limit,
    vector: vectors.get(idx).unwrap().clone(),
    collection_name: COLLECTION_NAME.to_string(),
    offset: Some(offset),
    score_threshold: Some(0.3),
    with_payload: Some(WithPayloadSelector {
        selector_options: Some(with_payload_selector::SelectorOptions::Enable(true)),
    }),
}
```

```user```
User: ${context.question}
Response:
```

---

### 11. Code Search Evaluation (hyde/code/evaluate.vm)

**Purpose:** Evaluates retrieved code snippets and answers questions about the codebase using the search results. Second phase of HyDE search.

**Location:** `/prompts/genius/en/hyde/code/evaluate.vm`

**Context Variables:**
- `${context.chatContext}` - Additional chat context and formatting rules
- `${context.language}` - Programming language of the code
- `${context.code}` - Retrieved code snippet
- `${context.question}` - User's question

**Usage:** After code snippets are retrieved, this prompt analyzes them and answers the user's question based on the found code.

**Prompt Template:**

```velocity
Your job is to answer a query about a codebase using the information above.

You must use the following formatting rules at all times:
- Provide only as much information and code as is necessary to answer the query and be concise
- If you do not have enough information needed to answer the query, do not make up an answer
- When referring to code, you must provide an example in a code block
- Keep number of quoted lines of code to a minimum when possible
- ${context.chatContext}
- Basic markdown is allowed
- If you have enough information, try your best to answer more details.
- You MUST provide key process of your thinking,

Code：

```${context.language}
${context.code}
```

Take a deep breath, and start to answer the question.

User's Question：${context.question}
```

---

### 12. Keyword Search Proposal (hyde/keywords/propose.vm)

**Purpose:** Converts natural language questions into relevant keywords for code search. Resolves pronouns and ambiguous terms, generates keyword variations.

**Location:** `/prompts/genius/en/hyde/keywords/propose.vm`

**Context Variables:**
- `${context.question}` - User's question

**Usage:** First phase of keyword-based HyDE search. Extracts and expands keywords from natural language queries for more effective code search.

**Prompt Template:**

```velocity
```system```
You are a coding assistant who helps the user answer questions about code in their workspace by providing a list of relevant keywords they can search for to answer the question.
The user will provide you with potentially relevant information from the workspace. This information may be incomplete.

DO NOT ask the user for additional information or clarification.
DO NOT try to answer the user's question directly.

**Additional Rules**
Think step by step:

1. Read the user's question to understand what they are asking about their workspace.
2. If the question contains pronouns such as 'it' or 'that', try to understand what the pronoun refers to by looking at the rest of the question and the conversation history.
3. If the question contains an ambiguous word such as 'this', try to understand what it refers to by looking at the rest of the question, the user's active selection, and the conversation history.
4. Output a precise version of the question that resolves all pronouns and ambiguous words like 'this' to the specific nouns they stand for. Be sure to preserve the exact meaning of the question by only changing ambiguous pronouns and words like 'this'.
5. Then output a short markdown list of up to 8 relevant keywords that the user could try searching for to answer their question. These keywords could be used as file names, symbol names, abbreviations, or comments in the relevant code. Put the most relevant keywords to the question first. Do not include overly generic keywords. Do not repeat keywords.
6. For each keyword in the markdown list of related keywords, if applicable add a comma-separated list of variations after it. For example, for 'encode', possible variations include 'encoding', 'encoded', 'encoder', 'encoders'. Consider synonyms and plural forms. Do not repeat variations.
7. keywords should be in English

**Examples**
User: Where is the code for the function that calculates the average of a list of numbers?

Response:
Where is calculating the average of a list of numbers?

- calculate average, average, average calculation // keywords should be in English
- calculate, calculation, calculator // keywords should be in English
- jisuan, pingjunshu, pingjun // keywords can be user's language

```user```
User: ${context.question}

Response:
```

---

### 13. Keyword Search Evaluation (hyde/keywords/evaluate.vm)

**Purpose:** Uses retrieved code to answer user questions. Provides guidance when information is insufficient.

**Location:** `/prompts/genius/en/hyde/keywords/evaluate.vm`

**Context Variables:**
- `${context.chatContext}` - Additional formatting rules
- `${context.language}` - Programming language
- `${context.code}` - Retrieved code
- `${context.question}` - User's question

**Usage:** Second phase of keyword-based search. Analyzes retrieved code and formulates answers.

**Prompt Template:**

```velocity
```system```
Use the above code to answer the following question. You should not reference any files outside of what is shown,
unless they are commonly known files, like a .gitignore or package.json. Reference the filenames whenever possible.
If there isn't enough information to answer the question, suggest where the user might look to learn more.

- ${context.chatContext}

```user```
User's Question: What is the output of the code?

code:

```${context.language}
${context.code}
```

User's Question: ${context.question}
```

---

## Page & UI Generation

This category includes prompts for generating frontend UI components based on requirements. Uses a two-phase approach: clarification and design.

### 14. Page Generation Clarification (page-gen-clarify.vm)

**Purpose:** Analyzes requirements and selects appropriate UI components from available frameworks. First phase of UI generation.

**Location:** `/prompts/genius/en/page/page-gen-clarify.vm`

**Context Variables:**
- `${context.frameworks}` - Frontend framework being used
- `${context.language}` - Programming language
- `${context.componentNames}` - Available component names
- `${context.pageNames}` - Available page templates
- `${context.requirement}` - User's requirements

**Usage:** When generating UI, first determines which components are needed based on requirements.

**Prompt Template:**

```velocity
You are a professional Frontend developer.
According to the user's requirements, you should choose the best components for the user in List.

- Framework: ${context.frameworks}
- Language: ${context.language}
- User component: ${context.componentNames}, ${context.pageNames}

For example:

- Question(requirements): Build a form for user to fill in their information.
- You should anwser: [Input, Select, Radio, Checkbox, Button, Form]

----

Here are the User requirements:

```markdown
${context.requirement}
```

Please choose the best Components for the user, just return the components names in a list, no explain.
```

---

### 15. Page Generation Design (page-gen-design.vm)

**Purpose:** Generates actual UI component code based on selected components and requirements. Second phase of UI generation.

**Location:** `/prompts/genius/en/page/page-gen-design.vm`

**Context Variables:**
- `${context.components}` - Component information and props
- `${context.requirement}` - User's requirements

**Usage:** After components are selected, generates the actual implementation code.

**Prompt Template:**

```velocity
You are a professional Frontend developer.
According to the user's requirements, and Components info, write Component for the user.

- User Components Infos: ${context.components}

For example:

- Question(requirements): Build a form for user to fill in their information.
// componentName: Form, props: { fields: [{name: 'name', type: 'text'}, {name: 'age', type: 'number'}] }
// componentName: Input, props: { name: 'name', type: 'text' }
// componentName: Input, props: { name: 'age', type: 'number' }
- Answer:
```react
<Form>
    <Input name="name" type="text" />
    <Input name="age" type="number" />
</Form>
```

----

Here are the requirements:

```markdown
${context.requirement}
```

Please write your code with Markdown syntax, no explanation is needed:
```

---

## SQL Generation

This category includes prompts for generating SQL queries based on natural language requirements. Uses a two-phase approach similar to UI generation.

### 16. SQL Generation Clarification (sql-gen-clarify.vm)

**Purpose:** Analyzes requirements and identifies which database tables are needed for the query. First phase of SQL generation.

**Location:** `/prompts/genius/en/sql/sql-gen-clarify.vm`

**Context Variables:**
- `${context.databaseVersion}` - Database type and version
- `${context.schemaName}` - Database schema name
- `${context.tableNames}` - Available table names
- `${context.requirement}` - User's requirements

**Usage:** When generating SQL, first determines which tables are relevant to the query.

**Prompt Template:**

```velocity
You are a professional Database Administrator.
According to the user's requirements, you should choose the best Tables for the user in List.

— User use database: ${context.databaseVersion}
- User schema name: ${context.schemaName}
- User tables: ${context.tableNames}

For example:

- Question(requirements): calculate the average trip length by subscriber type.// User tables: trips, users, subscriber_type
- You should anwser: [trips, subscriber_type]

----

Here are the User requirements:

```markdown
${context.requirement}
```

Please choose the best Tables for the user, just return the table names in a list, no explain.
```

---

### 17. SQL Generation Design (sql-gen-design.vm)

**Purpose:** Generates actual SQL query code based on selected tables and their schemas. Second phase of SQL generation.

**Location:** `/prompts/genius/en/sql/sql-gen-design.vm`

**Context Variables:**
- `${context.databaseVersion}` - Database type and version
- `${context.schemaName}` - Database schema name
- `${context.tableInfos}` - Table schemas and column information
- `${context.requirement}` - User's requirements

**Usage:** After tables are identified, generates the complete SQL query.

**Prompt Template:**

```velocity
You are a professional Database Administrator.
According to the user's requirements, and Tables info, write SQL for the user.

— User use database: ${context.databaseVersion}
- User schema name: ${context.schemaName}
- User tableInfos: ${context.tableInfos}

For example:

- Question(requirements): calculate the average trip length by subscriber type.
// table `subscriber_type`: average_trip_length: int, subscriber_type: string
- Answer:
```sql
select average_trip_length from subscriber_type where subscriber_type = 'subscriber'
```

----

Here are the requirements:

```markdown
${context.requirement}
```

Please write your SQL with Markdown syntax, no explanation is needed. :
```

---

## Infrastructure & DevOps

This category includes prompts for generating infrastructure and CI/CD configuration files.

### 18. Dockerfile Generation (generate-dockerfile.vm)

**Purpose:** Generates optimized Dockerfiles with multi-stage builds for containerizing applications.

**Location:** `/prompts/genius/en/sre/generate-dockerfile.vm`

**Context Variables:**
- `${context.buildContext}` - Build context and requirements

**Usage:** Creates production-ready Dockerfiles with separate build and runtime stages.

**Prompt Template:**

```velocity
Please write a Dockerfile with minimal steps.

${context.buildContext}
- I need the building to be done in separate base image than running the build
- I need the application port to be 3000

Output only the Dockerfile content without any explanation.
```

---

### 19. GitHub Actions Generation (generate-github-action.vm)

**Purpose:** Creates GitHub Actions workflow YAML files for building projects and running tests.

**Location:** `/prompts/genius/en/cicd/generate-github-action.vm`

**Context Variables:**
- `${context.buildContext}` - Build context and requirements

**Usage:** Generates CI/CD pipelines for automated testing and building.

**Prompt Template:**

```velocity
Create build.yml YAML file for GitHub Action for build project and runs tests

- ${context.buildContext}
- OS: latest ubuntu version
```

---

## Model & Reranking

This category contains prompts for LLM-based reranking of search results.

### 20. LLM Reranker (llm-reranker.vm)

**Purpose:** Evaluates relevance of code snippets to search queries. Returns binary Yes/No judgments for reranking search results.

**Location:** `/prompts/genius/en/model/reranker/llm-reranker.vm`

**Context Variables:**
- `${context.query}` - The search query
- `${context.documentId}` - Document/file path
- `${context.document}` - Code snippet content

**Usage:** Used in search pipelines to rerank retrieved results by relevance. Helps filter out irrelevant search results.

**Prompt Template:**

```velocity
You are an expert software developer responsible for helping detect whether the retrieved snippet of code is relevant to the query. For a given input, you need to output a single word: "Yes" or "No" indicating the retrieved snippet is relevant to the query.

Query: Where is the FastAPI server?
Snippet:
```/Users/andrew/Desktop/server/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {{"Hello": "World"}}
```
Relevant: Yes

Query: Where in the documentation does it talk about the UI?
Snippet:
```/Users/andrew/Projects/bubble_sort/src/lib.ts
function bubbleSort(arr: number[]): number[] {
    for (let i = 0; i < arr.length; i++) {
        for (let j = 0; j < arr.length - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
            }
        }
    }
    return arr;
}
```
Relevant: No

Query: ${context.query}
Snippet:
```${context.documentId}
${context.document}
```
Relevant:
```

---

## Quick Actions

This category contains prompts for quick code generation tasks.

### 21. Quick Action (quick-action.vm)

**Purpose:** Generates concise code snippets for quick tasks without extra explanation or comments.

**Location:** `/prompts/genius/en/quick/quick-action.vm`

**Context Variables:**
- `${context.task}` - The task description

**Usage:** For rapid code generation when users need quick snippets without verbose explanations.

**Prompt Template:**

```velocity
Generate a concise code snippet with no extra text, description, or comments.

The code should achieve the following task:

${context.task}
```

---
