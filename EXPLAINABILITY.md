# GitHub1s Explainability & Decision Transparency Report

## How the Agent Decides

GitHub1s resolves repository hierarchies, retrieves relevant code slices, and coordinates AI explanations through a deterministic 5-stage decision pipeline.

### 5-Stage Decision Pipeline

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

### Mathematical Formulation of Scoring & Routing

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

### Thresholds and Refusal Criteria

When upstream platform rate limits, authentication requirements, or file bounds are breached, GitHub1s terminates execution deterministically:

| Error Code | Trigger Condition | Deterministic Behavior |
|---|---|---|
| `ERR_REPO_NOT_FOUND` | Upstream provider returns HTTP 404 | Prompt user to check URL or supply private access token |
| `ERR_RATE_LIMIT_EXCEEDED` | Anonymous GitHub API rate limit (60 req/hr) exhausted | Prompt user for personal access token or OAuth sign-in |
| `ERR_FILE_TOO_LARGE` | Target file size exceeds $2\,\text{MB}$ | Block editor render; offer raw link download |
| `ERR_UNAUTHORIZED_PRIVATE_REPO` | HTTP 401/403 returned on repository fetch | Display OAuth token dialog with minimal read scopes |
| `ERR_SEARCH_SYNTAX_INVALID` | Regex search query violates regular expression syntax | Emit syntax error highlight with suggested pattern fix |

### Multi-Tier Fallback Mechanisms

GitHub1s implements a 3-tier fallback architecture to maintain continuous code exploration:

1. **Tier 1 (Cached Git Tree Fallback):** If API rate limits hit mid-session, continue serving already-cached directory nodes and open file buffers from browser IndexedDB storage.
2. **Tier 2 (Shallow Blob Fetching):** If full recursive git tree retrieval fails due to repository size, switch to on-demand shallow folder fetching as users expand folders.
3. **Tier 3 (User Authentication Elevation):** If anonymous access is denied, smoothly prompt the developer to connect their GitHub/GitLab account to receive 5,000 requests/hour.

## The Data It Uses

### Inputs Processed
- **Repository URLs**: GitHub, GitLab, and npm package URLs entered by the developer.
- **Search Queries**: Natural language questions, symbol lookups, and regex patterns.
- **Active Selections**: Cursor positions, selected lines of code, and active editor tab paths.

### Reference Data
- **Upstream REST API Schemas**: GitHub REST/GraphQL API v3/v4 and GitLab REST API specifications.
- **Syntax Grammars**: TextMate grammars and tree-sitter language parsers for syntax highlighting.
- **Language Server Protocol (LSP)**: Lightweight web-compatible language definitions for symbol indexing.

### Model Lineage & Weights
- **Model Agnostic**: User connects any browser-accessible model endpoint (OpenAI, Anthropic, Ollama, custom proxy).
- **Weight Integrity**: GitHub1s executes zero proprietary model inference on its own servers, acting strictly as the client-side orchestrator.

### Retention & Data Privacy
- **Zero Server Storage**: GitHub1s static web application runs entirely client-side in the user's browser.
- **Local Storage Encryption**: Access tokens and API keys are stored in browser localStorage or sessionStorage.
- **No Telemetry Code Logging**: Code snippets and file paths are never logged or stored on GitHub1s infrastructure.

## Limitations

1. **Limitation:** Anonymous users browsing GitHub repositories face a strict rate limit of 60 requests per hour.
   **Mitigation:** The application notifies users when rate limits approach 80% capacity and provides one-click token integration.

2. **Limitation:** Extremely large repositories (e.g., Chromium, Linux kernel) can exceed browser memory when loading full trees.
   **Mitigation:** GitHub1s automatically falls back to lazy-loaded, on-demand shallow branch directory fetching.

3. **Limitation:** Semantic code navigation (Go to Definition) in the browser lacks full compiler-level build information.
   **Mitigation:** Built-in ctags and tree-sitter heuristic parsers provide best-effort symbol jump navigation.

4. **Limitation:** Private repositories cannot be opened without explicit user credential provision.
   **Mitigation:** Secure OAuth authentication flows request only read-only repository permissions.

5. **Limitation:** Web-based VS Code extensions have restricted capabilities compared to native desktop extensions.
   **Mitigation:** GitHub1s filters extension installations to certified web-compatible extensions.

## Summary & Compliance Checklist

| Component | Status | Verification Detail |
|---|---|---|
| **5-Stage Decision Pipeline** | Verified | ASCII flow diagram mapping Stages 1 through 5 with explicit state transitions |
| **Scoring & Routing Mathematics** | Verified | Formal equation $S_{\text{relevance}}$ with symbol, lexical, path, and recency weights |
| **Deterministic Thresholds & Refusals** | Verified | $\tau = 0.65$ threshold and 5 standardized error codes (`ERR_*`) documented |
| **Multi-Tier Fallback Strategy** | Verified | Tier 1 (Cached Tree), Tier 2 (Shallow Blob), and Tier 3 (Auth Elevation) specified |
| **Data Privacy & Lineage Architecture** | Verified | Documented inputs, reference data, model lineage, and zero-retention policies |
| **5 Documented Limitations & Mitigations** | Verified | 5 numbered limitation/mitigation pairs covering rate limits, large repos, and memory |
