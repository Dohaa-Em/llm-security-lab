# Attack / defense matrix (OWASP LLM Top 10 2025)

| OWASP | Category | Tested on | Attack | Defense |
|---|---|---|---|---|
| LLM01 | Prompt Injection | Targets 1, 2 | Fiction jailbreak; hidden instruction in a RAG document | Input validator, hardened system prompt, document signing |
| LLM02 | Sensitive Information Disclosure | Target 1 | Credentials leaked through a story | Input and output validators |
| LLM04 | Data and Model Poisoning | Target 2 | Fake document injected into ChromaDB | NetworkPolicy, signature check, audit script |
| LLM05 | Improper Output Handling | Target 2 | XSS payload returned by the model | Output sanitization |
| LLM06 | Excessive Agency | Target 3 | `execute_shell`, ServiceAccount token theft | Tools disabled, RBAC, non-root, egress policy |
| LLM07 | System Prompt Leakage | Target 1 | Sentence completion, `[SYSTEM]` block | Hardened prompt, input validator |
| LLM09 | Misinformation | Target 2 | Poisoned "policy" returned as fact | Confidence score |
| LLM10 | Unbounded Consumption | Target 1 | 150,000-character payload, request flood | Size limit (10 KB), rate limit (5 req/min/IP) |

Note: some labels in the project's Kibana dashboards use earlier numbering. This table follows the official 2025 list.
