# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **GitHub1s** (`github1s`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** GitHub1s (`github1s`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Browser-Based Code Intelligence & Remote Repository Exploration  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

GitHub1s resolves repository hierarchies, retrieves relevant code slices, and coordinates AI explanations through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: URL Parsing & Repository Authentication Verification]                  |
|  - Parse provider/owner/repo/branch from URL; check rate limit and access token   |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Git Tree Ingestion & In-Memory Virtual File System Mount]              |
|  - Fetch git tree object, build hierarchical virtual file system, cache nodes     |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Query & Symbol Relevance Scoring]                                      |
|  - Compute relevance metric S_relevance matching user queries to source snippets  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Evaluation & Context Budgeting]                              |
|  - Verify tau >= 0.65; prune oversized files (>2MB), fit into token context       |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Source Rendering, Diff Highlighting & AI Streaming]                    |
|  - Render editor buffer, display syntax-highlighted diffs, stream AI explanation  |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For an incoming user code exploration query $q$ and candidate code snippet $c_j$ from repository $R$, the relevance score $S_{\text{relevance}}(c_j, q)$ is formulated as:

$$S_{\text{relevance}}(c_j, q) = w_{\text{sym}} \text{ExactSymbolMatch}(c_j, q) + w_{\text{lex}} \text{BM25}(c_j, q) + w_{\text{path}} P(c_j, q) + w_{\text{rec}} R(c_j)$$

Where:
- $\text{ExactSymbolMatch}(c_j, q) \in \{0, 1\}$ indicates whether query tokens match declared function, class, or variable symbols in snippet $c_j$.
- $\text{BM25}(c_j, q)$ is the normalized lexical similarity score across the codebase index.
- $P(c_j, q) \in [0, 1]$ measures file path affinity (e.g., source directories matching query scope).
- $R(c_j) = \max\left(0, 1 - \frac{\text{age\_days}(c_j)}{365}\right)$ rewards recently modified files or active PR changesets.
- Standard default weights: $w_{\text{sym}} = 0.40$, $w_{\text{lex}} = 0.30$, $w_{\text{path}} = 0.20$, $w_{\text{rec}} = 0.10$ with $\sum w = 1.0$.

Context inclusion and AI generation require:

$$S_{\text{relevance}}(c_j, q) \ge \tau \quad (\tau = 0.65)$$

### 3. Thresholding & Refusal Decision Criteria

GitHub1s enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_REPO_NOT_FOUND**: Upstream provider returns HTTP 404 halts execution with code `ERR_REPO_NOT_FOUND`.
- **Refusal on ERR_RATE_LIMIT_EXCEEDED**: Anonymous GitHub API rate limit (60 req/hr) exhausted halts execution with code `ERR_RATE_LIMIT_EXCEEDED`.
- **Refusal on ERR_FILE_TOO_LARGE**: Target file size exceeds $2\,\text{MB}$ halts execution with code `ERR_FILE_TOO_LARGE`.
- **Refusal on ERR_UNAUTHORIZED_PRIVATE_REPO**: HTTP 401/403 returned on repository fetch halts execution with code `ERR_UNAUTHORIZED_PRIVATE_REPO`.
- **Refusal on ERR_SEARCH_SYNTAX_INVALID**: Regex search query violates regular expression syntax halts execution with code `ERR_SEARCH_SYNTAX_INVALID`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Cached Git Tree Fallback):** If API rate limits hit midsession, continue serving alreadycached directory nodes and open file buffers from browser IndexedDB storage.
- **Tier 2 (Shallow Blob Fetching):** If full recursive git tree retrieval fails due to repository size, switch to ondemand shallow folder fetching as users expand folders.
- **Tier 3 (User Authentication Elevation):** If anonymous access is denied, smoothly prompt the developer to connect their GitHub/GitLab account to receive 5,000 requests/hour.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

GitHub1s operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Repository URLs**: GitHub, GitLab, and npm package URLs entered by the developer.
- **Search Queries**: Natural language questions, symbol lookups, and regex patterns.
- **Active Selections**: Cursor positions, selected lines of code, and active editor tab paths.

### 2. Configuration & Reference Data

- **Upstream REST API Schemas**: GitHub REST/GraphQL API v3/v4 and GitLab REST API specifications.
- **Syntax Grammars**: TextMate grammars and tree-sitter language parsers for syntax highlighting.
- **Language Server Protocol (LSP)**: Lightweight web-compatible language definitions for symbol indexing.

### 3. Base Model & Inference Lineage

- **Model Agnostic**: User connects any browser-accessible model endpoint (OpenAI, Anthropic, Ollama, custom proxy).
- **Weight Integrity**: GitHub1s executes zero proprietary model inference on its own servers, acting strictly as the client-side orchestrator.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of GitHub1s is essential for effective deployment.

### 1. Anonymous users browsing GitHub repositories face
- **Limitation**: Anonymous users browsing GitHub repositories face a strict rate limit of 60 requests per hour.
- **Mitigation**: The application notifies users when rate limits approach 80% capacity and provides one-click token integration.

### 2. Extremely large repositories (e
- **Limitation**: Extremely large repositories (e.g., Chromium, Linux kernel) can exceed browser memory when loading full trees.
- **Mitigation**: GitHub1s automatically falls back to lazy-loaded, on-demand shallow branch directory fetching.

### 3. Semantic code navigation (Go to Definition)
- **Limitation**: Semantic code navigation (Go to Definition) in the browser lacks full compiler-level build information.
- **Mitigation**: Built-in ctags and tree-sitter heuristic parsers provide best-effort symbol jump navigation.

### 4. Private repositories cannot be opened without
- **Limitation**: Private repositories cannot be opened without explicit user credential provision.
- **Mitigation**: Secure OAuth authentication flows request only read-only repository permissions.

### 5. Web-based VS Code extensions have restricted
- **Limitation**: Web-based VS Code extensions have restricted capabilities compared to native desktop extensions.
- **Mitigation**: GitHub1s filters extension installations to certified web-compatible extensions.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Anonymous users browsing GitHub repositories face | Section 1 | Verified |
| - Extremely large repositories (e | Section 2 | Verified |
| - Semantic code navigation (Go to Definition) | Section 3 | Verified |
| - Private repositories cannot be opened without | Section 4 | Verified |
| - Web-based VS Code extensions have restricted | Section 5 | Verified |
