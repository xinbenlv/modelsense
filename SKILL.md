---
name: modelsense
description: >
  ModelSense — The right model for the right job. Recommends the best LLM
  model and effort level for any task, based on benchmark data, task analysis,
  and the user's configured providers. Use when the user asks "which model
  should I use?", "what's the best model for X?", or wants help choosing
  between models/effort levels.
user-invocable: true
---

# ModelSense Skill

## Purpose

ModelSense helps users pick the optimal model and effort level for their task.
It does NOT route automatically on every request (use a provider plugin for that).
It's an on-demand advisor: ask it a question, get a clear recommendation with reasoning.

## When to trigger

- User asks: "which model for X?", "should I use Opus or Sonnet?", "what effort level?"
- User wants to understand what a benchmark means
- User wants ModelSense to auto-switch the session model

## Inputs to collect (infer from context, ask only if truly unclear)

1. **Task description** — what is the user trying to do?
2. **Effort preference** (optional): `quick` / `balanced` / `deep` / `research`
   - If not specified, infer from task urgency/complexity
3. **Auto-switch?** — does the user want ModelSense to apply the recommendation automatically?

## Recommendation Process

### Step 1 — Task Analysis

Classify the task across these dimensions:
- **Domain**: code, math, reasoning, writing, dialogue, document analysis, multimodal, research
- **Complexity**: simple / moderate / complex / research-grade
- **Output type**: text, code, JSON, long-form, structured data
- **Context length needed**: short (<8K), medium (8–32K), long (32K+), very long (100K+)
- **Special requirements**: function calling, thinking/CoT, multimodal, speed-sensitive

### Step 2 — Benchmark Matching

Cross-reference task domain with relevant benchmarks from `data/benchmarks.yaml`.

| Benchmark | Best for |
|-----------|----------|
| HumanEval / SWE-bench | Code generation, debugging, engineering |
| GPQA | Graduate-level science & research |
| MATH / AIME | Mathematical reasoning |
| MMLU | General knowledge, multidomain QA |
| Needle-in-Haystack | Long-context retrieval |
| MT-Bench / Arena Elo | Dialogue, writing quality |
| BBH (Big-Bench Hard) | Complex reasoning, multi-step logic |

### Step 3 — Effort × Model Matrix

| Effort | Target quality | Typical model tier |
|--------|---------------|-------------------|
| `quick` | Good enough, fast | Haiku / Flash / GLM |
| `balanced` | High quality, reasonable cost | Sonnet / GPT-4o |
| `deep` | Best available, thinking on | Opus / o3 |
| `research` | No cost limit, maximum quality | Opus + thinking=high |

### Step 4 — Provider Filter

Check the user's available providers:
- Run: `openclaw models list` via exec tool (or read from context)
- Only recommend models the user can actually use
- Flag when a top pick requires a provider they haven't configured

### Step 5 — Output the Recommendation

Use markdown headers, bold, and tables for clean rendering in terminal UIs (Claude Code, Codex CLI, Cursor). No emojis.

Format:

```markdown
### Recommendation

**Model:** `<model>`
**Effort:** `<level>`
**Why:** <1-2 sentence benchmark-grounded rationale>
**Settings:** <thinking level, function calling, etc.>
**Cost:** <rough $/M or relative>

#### Key Benchmarks

| Benchmark | Recommended | Score | Runner-up | Score |
|-----------|-------------|-------|-----------|-------|
| <name>    | <model>     | <score> | <model> | <score> |
| <name>    | <model>     | <score> | <model> | <score> |

> Show 2-4 most relevant benchmarks from `data/benchmarks.yaml` for this task.

#### Alternatives

| Model | Trade-off |
|-------|-----------|
| `<model B>` | Faster / cheaper |
| `<model C>` | Higher quality |
```

**IMPORTANT**: Always include the "Key Benchmarks" table. Select the 2-4 benchmarks
from `data/benchmarks.yaml` most relevant to the user's task domain (see the Benchmark Matching
table in Step 2). Show the actual scores for the recommended model and at least one competitor.
This makes the recommendation transparent and verifiable.

## Auto-Switch Behaviors

### Option A: Advisory only (default)
Just output the recommendation. Tell user: "Run `/model <name>` to switch."

### Option B: Switch current session
If user confirms or says "yes switch" / "apply it":
```python
session_status(model="<provider/model>")
```
Notify user: "Switched to X for this session. Run `/model default` to reset."

### Option C: Delegate task to best model
If user says "just do it with the best model":
```python
sessions_spawn(
  task="<original task>",
  model="<recommended model>",
  thinking="<level>"
)
```

## Data Files

- `data/benchmarks.yaml` — benchmark definitions, score leaders, task mappings
- `data/models.yaml` — model catalog (updated via GitHub Actions weekly)

## Examples

**User**: "I need to write a Solidity audit report"
→ Domain: code + security + long-form
→ Benchmarks: SWE-bench, HumanEval
→ Recommendation: `claude-opus-4-6` with `thinking=high`, effort=`deep`
→ Key benchmarks:
  - SWE-bench Verified: Opus 72.5% · GPT-5.2 69.1%
  - HumanEval: Opus 96.4% · GPT-5.2 95.8%

**User**: "Quick summary of this Slack thread"
→ Domain: dialogue, short
→ Recommendation: `claude-haiku-4-5` or `gemini-flash`, effort=`quick`
→ Key benchmarks:
  - MT-Bench: Opus 9.6/10 · Sonnet 9.3/10 (Haiku fast enough for summaries)

**User**: "Prove this mathematical conjecture"
→ Domain: math, research-grade
→ Benchmarks: MATH, AIME, GPQA
→ Recommendation: `o3` or `claude-opus-4-6` with `thinking=high`, effort=`research`
→ Key benchmarks:
  - MATH: o3 97.9% · Opus 95.1%
  - GPQA Diamond: o3 87.7% · Opus 83.2%
  - BBH: o3 92.4% · Opus 89.7%
