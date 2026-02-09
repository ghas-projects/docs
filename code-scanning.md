# Code Scanning 

## Introduction - What is Code Scanning 

- Static analysis examines code for issues without executing it 

- GitHub code scanning automates source code checks to catch vulnerabilities early
 
- Alerts appear in a dedicated tab and on pull requests in GitHub

- CodeQL powers GitHub’s code scanning with semantic analysis


<img width="1973" height="1024" alt="image" src="https://github.com/user-attachments/assets/dd08ec97-dc5a-4f7f-8e00-6c46621be35c" />

## What is CodeQL 

CodeQL analyzes code by treating it as data. It does this by:
1. Creating a database that represents the codebase and its relationships.
2. Running queries against that database to identify vulnerabilities, code smells, and other patterns.

There are three primary ways to run CodeQL:

1. **Default setup**  
   Automatically detects languages, selects query suites, and configures triggers.

2. **Advanced setup**  
   Uses a custom GitHub Actions workflow, giving you fine-grained control over languages, queries, and build steps.

3. **CodeQL CLI in an external CI/CD pipeline**  
   Runs CodeQL outside of GitHub Actions, with results uploaded back to GitHub for code scanning.


## Step 1: Creating a Database 

The first step in any CodeQL analysis is creating a **CodeQL database**.  
The database contains a structured representation of the source code, which is later queried during analysis.

Regardless of whether you are using GitHub Actions (Default or Advanced setup) or a third-party CI/CD system, the **CodeQL CLI** is required. The CLI provides the commands needed to create databases, run analyses, and manage results.

You can find the full CLI reference here:  
https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual

### Database Initialization

The database initialization step is performed by the following GitHub Action:

```yaml
github/codeql-action/init@vX
```

This action invokes the equivalent of the `codeql database init` CLI command.

**Purpose of this step:**

- Creates the directory structure for a CodeQL database
- Configures language-specific extraction settings
- Prepares the database to receive extracted code data
- Does *not* yet populate the database with a raw QL dataset

### Example CLI Command (Default Setup)

```bash
codeql database init \
  --force-overwrite \
  --db-cluster ${DIR} \
  --source-root=${SOURCE_DIR} \
  --calculate-language-specific-baseline \
  --sublanguage-file-coverage \
  --extractor-include-aliases \
  --language={LANGUAGE} \
  --codescanning-config=/home/runner/work/_temp/user-config.yaml \
  --build-mode=none
```
## Build Modes for Compiled Languages

### `none`

- No build is performed
- Supported for all interpreted languages
- Also supported for **C/C++**, **C#**, **Java**, and **Rust**

### `autobuild`

- CodeQL automatically detects and executes the build
- Supported for:
  - **C/C++**
  - **C#**
  - **Go**
  - **Java**
  - **Kotlin**
  - **Swift**

### `manual`

- Build steps are explicitly defined in the workflow
- Recommended for complex or non-standard builds
- Supported for:
  - **C/C++**
  - **C#**
  - **Go**
  - **Java**
  - **Kotlin**
  - **Swift**


Query suites: code scanning (default), security-extended, and security-and-quality
