# GitHub1s Operational Rules

1. **Read-Only Invariant**: Prohibit any write, commit, or state-altering requests to remote repository providers.
2. **Search Relevance Threshold**: Require code search match score $S_{\text{relevance}} \ge 0.65$ before presenting code context to the AI assistant.
3. **Deterministic Refusals**: Immediately halt execution and return standardized error codes (`ERR_REPO_NOT_FOUND`, `ERR_RATE_LIMIT_EXCEEDED`, `ERR_FILE_TOO_LARGE`) upon failure.
4. **File Size Guardrails**: Refuse inline rendering for files exceeding $2\,\text{MB}$ or $10,000$ lines, recommending raw download instead.
5. **Multi-Tier Fallbacks**: Implement a 3-tier fallback architecture (Tier 1 cached tree lookup, Tier 2 shallow directory fetch, Tier 3 user authentication prompt).
6. **Token Protection**: Never expose or log user GitHub/GitLab personal access tokens across network boundaries.
