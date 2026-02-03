## Local LLM Tuning with LM Studio (OpenClaw as Agent Brain)

When running OpenClaw with a locally hosted LLM (for example via LM Studio using an OpenAI-compatible API), correct parameter tuning is critical for:

- stability  
- latency  
- memory usage (KV cache)  
- output quality  
- avoiding runaway generations  

This section summarizes best-practice settings and trade-offs for local LLM deployments.

---

### 1. Context Window (`contextWindow`)

The context window primarily impacts **RAM usage via the KV cache**.

**Guidelines:**
- Start with **8k context**
- If stable and responsive, increase to **12k–16k**
- Only use very large contexts (e.g. 128k+) if:
  - memory headroom is verified
  - swap is not used
  - latency remains acceptable

**Key insight:**
> A large context window is only useful if it is filled with meaningful information.  
> Excessively large windows increase memory pressure and slow down inference.

**Rule of thumb:**  
Prefer *moderate context + good summarization* over extreme context sizes.

---

### 2. Max Tokens per Response (`maxTokens`)

This limits how long a single model response can become.

**Recommended defaults:**
- **600–1000 tokens** for normal tasks  
- **1000–2000 tokens** for coding tasks  
- Avoid very large values (e.g. 8000+) unless strictly necessary

**Why this matters:**
- Prevents endless outputs  
- Reduces risk of tool loops  
- Improves responsiveness  
- Encourages multi-step reasoning instead of single massive replies  

**Best practice:**  
Prefer multiple short iterations over one extremely long response.

---

### 3. Temperature and Sampling

For agent-style workloads (coding, tools, analysis):

| Parameter | Typical Range | Purpose |
|-----------|----------------|---------|
| temperature | 0.2 – 0.6 | Lower = more deterministic |
| top_p | 0.8 – 0.95 | Limits randomness |
| repeat penalty | 1.05 – 1.15 | Avoids loops |

Lower temperature is recommended for code and factual tasks.  
Higher temperature increases creativity but also error risk.

---

### 4. Tool Output Management (Context Hygiene)

Tool outputs (HTML, logs, API responses) can quickly consume useful context.

**Recommended measures:**
- Enable context pruning or summarization  
- Truncate large tool outputs  
- Prefer structured extraction over raw dumps  

**Why:**  
Large unfiltered tool outputs consume context without improving reasoning quality.

---

### 5. Context Pruning / Session Compaction

For long-running sessions, enable automatic cleanup of old tool results.

Typical strategy:
- Soft-trim long tool outputs  
- Hard-clear very old tool results  
- Keep only summaries and conclusions  

This preserves:
- task continuity  
- memory efficiency  
- reasoning quality  

---

### 6. OpenClaw Model Binding (Example Pattern)

```jsonc
{
  "models": {
    "mode": "merge",
    "providers": {
      "lmstudio": {
        "baseUrl": "http://127.0.0.1:1234/v1",
        "api": "openai-completions",
        "models": [
          {
            "id": "MODEL_ID_FROM_LM_STUDIO",
            "name": "Local LLM via LM Studio",
            "contextWindow": 8192,
            "maxTokens": 1000
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "lmstudio/MODEL_ID_FROM_LM_STUDIO"
      }
    }
  }
}
```

---

### 7. Large Context Strategy (Advanced)

If very large context windows are supported by your hardware:

**Best practice:**
- Keep a large `contextWindow`
- Lower `maxTokens` (e.g. 1500–2000)
- Enforce:
  - tool-output trimming  
  - summarization  
  - structured responses  

This allows:
- long documents  
- complex research tasks  

without:
- excessive latency  
- runaway outputs  

---

### 8. Concurrency (`maxConcurrent`)

Parallel requests multiply memory usage and latency.

**Recommended values for local LLMs:**

| Setting | Typical Value |
|---------|----------------|
| maxConcurrent | 1–2 |
| subagents.maxConcurrent | 2–4 |

**Why:**  
Each concurrent generation maintains its own KV cache and runtime state.  
High concurrency causes memory spikes and slowdowns instead of linear speedup.

**Rule:**  
Prefer fewer concurrent tasks with higher throughput per task.

---

### 9. Improving Code Quality and Source Accuracy (Prompting Strategy)

For reliable agent behavior:

**Best practices:**
- Enforce structured answers:
  - plan → implementation → tests → review  
- Require explicit disambiguation of numeric values:
  - population vs sample vs respondents  
- Require citation or quotation for extracted numbers  
- Prefer labeled lists over single values  

This reduces:
- misinterpretation of sources  
- hallucinated numbers  
- low-quality code output  

---

## Summary

For local LLM setups:

- Optimize for stability first  
- Limit output size  
- Manage tool noise  
- Keep concurrency low  
- Use summarization  
- Favor structured reasoning  
- Measure memory usage, not theoretical limits  

> A well-tuned local model often outperforms a larger but poorly managed one.
