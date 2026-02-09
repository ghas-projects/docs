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

### Tool Setup

You should always download the **CodeQL bundle** from the **Releases** page of the [`codeql-action` repository](https://github.com/github/codeql-action/releases).  
The bundle includes the CodeQL CLI along with a set of queries that are guaranteed to be compatible with that release.

Downloading the CodeQL CLI and query packs separately is **not recommended**, as mismatched versions can lead to incompatibilities and unexpected analysis behavior.

The source code for the CodeQL GitHub Action itself is available here:  
https://github.com/github/codeql-action/

You can view the CodeQL queries here:
https://github.com/github/codeql

---
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

All supported input options are declared in the [`init` action.yml](https://github.com/github/codeql-action/blob/main/init/action.yml). This file is also the entry point for the action implementation.

This action invokes the equivalent of the [`codeql database init`](https://docs.github.com/en/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-init) CLI command.

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
---
## Step 2: Tracing the Build 

### Build Modes

#### `none`

- No build is performed
- Supported for all interpreted languages
- Also supported for **C/C++**, **C#**, **Java**, and **Rust**

#### `autobuild`

- CodeQL automatically detects and executes the build
- Supported for:
  - **C/C++**
  - **C#**
  - **Go**
  - **Java**
  - **Kotlin**
  - **Swift**

#### `manual`

- Build steps are explicitly defined in the workflow
- Recommended for complex or non-standard builds
- Supported for:
  - **C/C++**
  - **C#**
  - **Go**
  - **Java**
  - **Kotlin**
  - **Swift**

---

### Autobuild in GitHub Actions

In GitHub Actions, [autobuild](https://github.com/github/codeql-action/blob/main/autobuild/action.yml) is triggered using:

```yaml
github/codeql-action/autobuild@v4
```

This invokes [`codeql database trace-command`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-trace-command), which runs build commands under a tracer and seeds the database with extracted data.Note - it does not finalize the database.

```bash
codeql database trace-command \
  --use-build-mode \
  --working-dir {DIR} \
  {DATABASE}
```

---

## Step 3: Performing CodeQL Analysis

In GitHub Actions, analysis and database finalization are performed using :

```yaml
github/codeql-action/analyze@v4
```

The supported input options and implementation details for this action are defined in the [analyze action.yml](https://github.com/github/codeql-action/blob/main/analyze/action.yml).

### Database Finalization

This invokes the [`codeql database finalize`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-finalize) command. This finalizes a database that was created with `codeql database init` and subsequently seeded with analysis data using `codeql database trace` command. This needs to happen before the new database can be queried.

```bash
codeql database finalize \
  --finalize-dataset \
  --threads=2 \
  --ram=6915 \
  {DATABASE}
```

Finalization is required before queries can be executed.

### Running Queries

The `[codeql database run-queries`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-run-queries) command runs one or more queries against a CodeQL database, saving the results to the results subdirectory of the database directory.

```bash
codeql database run-queries \
  --ram=6915 \
  --threads=2 \
  --expect-discarded-cache \
  --min-disk-free=1024 \
  -v \
  {DATABASE}
```

Built-in query suites include:

- **code-scanning** (default)
- **security-extended**
- **security-and-quality**

[[Documentation Link]](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-queries/about-built-in-queries)

---

## Step 4: Clean up and Bundling



### Cleanup

The [`codeql database cleanup`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-cleanup) command compacts a CodeQL database on disk.

Delete temporary data, and generally make a database as small as possible on disk without degrading its future usefulness

```bash
codeql database cleanup {DATABASE} --cache-cleanup=clear
```

### Bundle Database

The [`codeql database bundle`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-bundle) command creates a relocatable archive of a CodeQL database.

A command that zips up the useful parts of the database. This will only include the mandatory components, unless the user specifically requests that results, logs, TRAP, or similar should be included.


```bash
codeql database bundle {DATABASE} --output={DATABASE}.zip --name={NAME}
```

---

## The Common Case: Minimal CodeQL Commands

When running CodeQL locally or configuring a custom CI/CD workflow, you typically do **not** need to use the low-level “plumbing” commands directly.

GitHub Actions exposes these lower-level commands to support the wide variety of repository structures and build systems that exist in the ecosystem. However, for most standard workflows, CodeQL provides higher-level commands that encapsulate this complexity.

In the majority of cases, only two commands are required:

- [`codeql database create`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-create). This command creates, traces build and finalises the database. 
- [`codeql database analyze`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-analyze). This command runs the queries and produces SARIF results.

These commands handle database initialization, build tracing, extraction, finalization, and query execution under the hood.

The analysis step produces a SARIF file, which can then be uploaded to GitHub using the following action:

```yaml
github/codeql-action/upload-sarif@vX
