## Local LLM Tuning with LM Studio (OpenClaw as Agent Brain)

When running OpenClaw with a locally hosted LLM (for example via LM Studio using an OpenAI-compatible API), correct parameter tuning is critical for:

- stability
- latency
- memory usage (KV cache)
- output quality
- avoiding runaway generations

This section summarizes best-practice settings and trade-offs for local LLM deployments.

---

### 0. Docker Networking Impact (Critical!)

**This is often the biggest performance factor.**

Docker bridge networking adds significant latency for API calls to external services (like your LLM server):

| Network Mode | First Request | Subsequent Requests |
|--------------|---------------|---------------------|
| Bridge (`gateway-net`) | ~1.8s | ~0.01s |
| Host (`network_mode: host`) | ~0.3s | ~0.01s |

**Recommendation:** Use `network_mode: host` in your docker-compose.yml for 5-6x faster initial response times.

```yaml
services:
  openclaw-gateway:
    network_mode: host
    # ports section not needed with host networking
```

With host networking, the container shares the host's network stack, eliminating NAT overhead.

**Security note:** Use `--bind loopback` to ensure the gateway only listens on 127.0.0.1, then access via Cloudflare Tunnel or SSH tunnel.

---

### 1. Context Window (`contextWindow`)

The context window primarily impacts **RAM usage via the KV cache**.

**Guidelines:**
- Start with **16k–32k context** for modern models
- For large models (70B+), **32k** is a good balance
- Only use very large contexts (e.g. 100k+) if:
  - memory headroom is verified
  - swap is not used
  - latency remains acceptable

**Key insight:**
> A large context window is only useful if it is filled with meaningful information.
> Excessively large windows increase memory pressure and slow down inference.

**Recommended setting for 80B models:**
```json
"contextWindow": 32768
```

This provides enough context for complex tasks while avoiding excessive memory usage.

**Rule of thumb:**
Prefer *moderate context + good pruning* over extreme context sizes.

**Measure, don't guess:** Watch memory + swap (Activity Monitor / LM Studio stats). If swap increases or tokens/sec collapses under load, reduce `contextWindow` or concurrency.

---

### 2. Max Tokens per Response (`maxTokens`)

This limits how long a single model response can become.

**Recommended defaults:**
- **1024–2048 tokens** for general agent tasks
- **2048 tokens** for coding and complex reasoning
- Avoid very large values (e.g. 8000+) unless strictly necessary

**Recommended setting:**
```json
"maxTokens": 2048
```

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

Note: Repeat penalty is engine-dependent. Start low (≈1.05). If code formatting degrades, disable it.


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

Example (OpenClaw): enable aggressive pruning for faster response times:

```jsonc
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "2m",
        "tools": {
          "allow": ["exec", "read", "web_fetch", "browser"],
          "deny": ["*image*"]
        },
        "softTrim": { "maxChars": 2000, "headChars": 800, "tailChars": 800 },
        "hardClear": { "enabled": true, "placeholder": "[cleared]" }
      }
    }
  }
}
```

**Optimized settings explained:**
- **TTL 2m** (vs 5m): Faster cleanup of stale tool outputs
- **softTrim 2000 chars** (vs 4000): Smaller retained context per tool result
- **Shorter placeholder**: Minimal overhead for cleared content
- **tools.deny `*image*`**: Skip caching image-related tool outputs

---

### 6. OpenClaw Model Binding (Example Pattern)

```jsonc
{
  "models": {
    "mode": "merge",
    "providers": {
      "lmstudio": {
        "baseUrl": "http://LLM_SERVER_IP:1234/v1",  // Your LM Studio server
        "apiKey": "lmstudio-local",
        "api": "openai-responses",
        "models": [
          {
            "id": "your-model/model-name",
            "name": "Local LLM via LM Studio",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 32768,
            "maxTokens": 2048
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "lmstudio/your-model/model-name",
        "fallbacks": ["openai/gpt-4o-mini"]  // Optional cloud fallback
      }
    }
  }
}
```

**Notes:**
- Use `"api": "openai-responses"` for newer LM Studio versions
- Set `"cost": { ... }` to 0 for local models (free inference)
- Add cloud fallbacks for reliability when local LLM is unavailable

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

1. **Use host networking** – eliminates 5-6x latency penalty from Docker bridge
2. **Optimize context window** – 32k is a good balance for large models
3. **Set reasonable maxTokens** – 2048 for coding tasks
4. **Enable aggressive pruning** – 2min TTL, smaller softTrim values
5. **Keep concurrency low** – 1-2 for main agent, 2-4 for subagents
6. **Measure, don't guess** – monitor memory, swap, and tokens/sec

> A well-tuned local model often outperforms a larger but poorly managed one.

---

## Quick Reference: Optimized Settings

```jsonc
{
  "models": {
    "providers": {
      "lmstudio": {
        "baseUrl": "http://LLM_SERVER_IP:1234/v1",
        "api": "openai-responses",
        "models": [{
          "id": "your-model/model-name",
          "contextWindow": 32768,
          "maxTokens": 2048
        }]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "lmstudio/your-model/model-name" },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "2m",
        "softTrim": { "maxChars": 2000, "headChars": 800, "tailChars": 800 },
        "hardClear": { "enabled": true, "placeholder": "[cleared]" }
      },
      "maxConcurrent": 2,
      "subagents": { "maxConcurrent": 4 }
    }
  }
}
```

**docker-compose.yml:**
```yaml
services:
  openclaw-gateway:
    network_mode: host  # Critical for LLM performance!
```
