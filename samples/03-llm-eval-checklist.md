# A Practical Checklist for Evaluating LLM Features in a Product

Shipping an LLM feature is easy. Knowing whether it is *good enough* for real users—fast enough, trustworthy enough, cheap enough—is harder. Benchmark leaderboards rarely match your domain, and "vibes-based" QA does not survive a messy production prompt. This article is an opinionated howto: a concrete checklist for evaluating LLM features inside a product, plus a tiny Python scoring sketch you can adapt before you invest in a full eval platform.

The stance: treat LLM behavior like any other unreliable dependency. Measure it. Gate releases on metrics that map to user harm and cost. Prefer small, high-quality golden sets over giant noisy ones.

## What "evaluation" means in a product context

Research evals often optimize accuracy on public datasets. Product evals ask different questions:

- Does this answer the **user's** question with **our** data?
- Is latency acceptable on the p95 path users feel?
- Will this month's traffic blow the budget?
- When it fails, does it fail safely (refusal, citation miss, escalation)?

If your eval suite cannot detect a regression in those dimensions, it is theater.

## The checklist

Use this as a pre-launch and pre-prompt-change gate. Not every item needs a fancy dashboard on day one; each needs a named owner and a written threshold.

### 1. Define the user job and failure modes

Write one paragraph: who uses the feature, what success looks like, and the top three failure modes (for example: hallucinated policy, outdated pricing, leaked private notes). Eval design follows failure modes. If "sounds fluent" is your only success criterion, you will ship fluent wrong answers.

### 2. Build a golden set from real traffic (sanitized)

Collect 50–200 examples from production or dogfood, redacting PII. Include:

- Common happy paths
- Ambiguous asks
- Adversarial or out-of-scope prompts
- Cases that previously broke

Label an expected behavior per case: exact string (rare), required facts, required citations, or "should refuse." Golden sets beat synthetic-only sets because they capture your distribution.

### 3. Groundedness and faithfulness

For RAG or tool-using features, score whether claims are supported by retrieved context. A simple rubric:

- **Supported** — claim appears in or is directly entailed by context
- **Unsupported** — plausible but not in context
- **Contradicted** — conflicts with context

Opinion: do not rely solely on the same model family to grade itself without spot checks. Use a graded sample plus periodic human review. Public discussions of faithfulness (for example in RAG survey literature and vendor eval docs) converge on the same idea: separate **retrieval quality** from **generation faithfulness**.

Measure retrieval separately: was the right document in the top-k? A perfect generator cannot save missing context.

### 4. Latency budgets users feel

Track:

- Time to first token (TTFT) for streaming UIs
- End-to-end time to final answer
- p50 / p95 / p99, not just averages

Set budgets as product requirements ("p95 under 3s for support draft") and fail the eval when a prompt or model change violates them. A 2-point quality gain that doubles p95 is often a net product loss.

### 5. Cost per successful task

Log tokens (prompt + completion) and multiply by your contracted rates. Report **cost per successful task**, not only cost per call—retries, tool loops, and "try again with more context" inflate real spend.

Opinion: put a soft budget in CI for offline eval runs so a runaway agent loop cannot burn money unnoticed during experimentation.

### 6. Safety and policy regression

Even internal tools need checks: PII echo, prompt injection via retrieved documents, toxic or disallowed content. Keep a small dedicated suite. Run it on every prompt/model change. Automated classifiers help; humans still review borderline cases.

### 7. Determinism where you need it

For grading and CI, fix temperature (often 0) and record model version strings. Nondeterminism is fine in production UX experiments; it is painful in regression tests. Store raw outputs alongside scores so you can debug flakes.

### 8. Human review cadence

Automate first-pass metrics; schedule human review on a sample weekly or per release. Automation drifts. Humans catch new failure modes your rubric never named.

## A tiny Python scoring sketch

The following sketch scores a batch of offline results for latency, rough cost, and a naive groundedness heuristic (claim substring overlap with context). It is intentionally small—replace the groundedness function with a proper NLI/judge model when you outgrow it.

```python
from __future__ import annotations

from dataclasses import dataclass
from statistics import mean, quantiles


@dataclass
class EvalExample:
    example_id: str
    context: str
    output: str
    latency_ms: float
    prompt_tokens: int
    completion_tokens: int
    must_include: list[str]  # required facts / phrases for this case


PRICE_PROMPT_PER_1K = 0.005
PRICE_COMPLETION_PER_1K = 0.015


def groundedness_score(context: str, output: str, claims: list[str]) -> float:
    """Fraction of required claims whose normalized text appears in context AND output.

    This is a stand-in heuristic for demos and CI smoke tests—not a substitute
    for entailment models or human labels on high-risk domains.
    """
    if not claims:
        return 1.0
    ctx = context.lower()
    out = output.lower()
    hits = 0
    for claim in claims:
        c = claim.lower().strip()
        if c in out and c in ctx:
            hits += 1
        elif c in out and c not in ctx:
            # Unsupported claim present in output
            pass
    return hits / len(claims)


def cost_usd(prompt_tokens: int, completion_tokens: int) -> float:
    return (
        prompt_tokens / 1000.0 * PRICE_PROMPT_PER_1K
        + completion_tokens / 1000.0 * PRICE_COMPLETION_PER_1K
    )


def summarize(examples: list[EvalExample]) -> dict:
    latencies = [e.latency_ms for e in examples]
    grounded = [groundedness_score(e.context, e.output, e.must_include) for e in examples]
    costs = [cost_usd(e.prompt_tokens, e.completion_tokens) for e in examples]

    p95 = quantiles(latencies, n=20)[18] if len(latencies) >= 20 else max(latencies)

    return {
        "n": len(examples),
        "groundedness_mean": round(mean(grounded), 3),
        "latency_ms_mean": round(mean(latencies), 1),
        "latency_ms_p95": round(p95, 1),
        "cost_usd_mean": round(mean(costs), 5),
        "cost_usd_total": round(sum(costs), 5),
    }


def passes_gate(summary: dict) -> bool:
    return (
        summary["groundedness_mean"] >= 0.85
        and summary["latency_ms_p95"] <= 3000
        and summary["cost_usd_mean"] <= 0.02
    )


if __name__ == "__main__":
    batch = [
        EvalExample(
            example_id="support-001",
            context="Refunds are available within 30 days of purchase with a receipt.",
            output="You can request a refund within 30 days of purchase with a receipt.",
            latency_ms=820.0,
            prompt_tokens=400,
            completion_tokens=60,
            must_include=["30 days", "receipt"],
        ),
        EvalExample(
            example_id="support-002",
            context="Refunds are available within 30 days of purchase with a receipt.",
            output="All purchases include lifetime refunds with no receipt required.",
            latency_ms=910.0,
            prompt_tokens=400,
            completion_tokens=55,
            must_include=["30 days", "receipt"],
        ),
    ]
    summary = summarize(batch)
    print(summary)
    print("GATE_PASS" if passes_gate(summary) else "GATE_FAIL")
```

Run it:

```bash
python eval_sketch.py
```

You should see a mean groundedness below the gate (because `support-002` invents policy), which is exactly the point: the suite fails closed when claims leave the context.

### Extending the sketch

- Swap `must_include` for structured rubrics (JSON fields the model must populate).
- Add a retrieval hit-at-k metric when context comes from a vector store.
- Persist results as JSONL keyed by `model`, `prompt_version`, and git SHA.
- In CI, fail the job when `passes_gate` is false on the golden set.

## Release process that respects the checklist

1. Change prompt, model, tools, or retrieval → run golden set offline.
2. Compare groundedness, p95 latency, and cost to the previous baseline.
3. Spot-check new failures with a human.
4. Shadow traffic in production if the risk warrants it.
5. Only then flip the default.

Opinion: prompt PRs without eval artifacts should be treated like application PRs without tests—mergeable only with an explicit risk acceptance.

## Takeaways

- Product LLM eval is about **user jobs, failure modes, latency, groundedness, and cost**—not leaderboard bragging rights.
- Start with a **small golden set** from real (sanitized) traffic and hard gates.
- Separate **retrieval** quality from **generation** faithfulness.
- Track **p95 latency** and **cost per successful task**.
- Keep a tiny automated scorer early; graduate to better judges without abandoning human review.
- Gate releases on regressions the same way you would for an API contract.

You do not need a perfect eval platform to ship responsibly. You need a written checklist, a golden set that reflects your users, and the discipline to fail the build when groundedness, latency, or cost drifts past the line you drew.

