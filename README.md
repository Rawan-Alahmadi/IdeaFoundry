# IdeaFoundry

**An idea-validation analyst team for first-time founders in Saudi Arabia.**

A founder submits an early-stage business idea. A **Supervisor** agent reads it, decides by
calling a real handoff tool, which specialist analysts the idea actually needs,
**pauses for the founder to confirm**, then runs those analysts in parallel.
They ground their work in official **Monsha'at** guidance and live **web search**, and label
every factual statement as *evidence* (with a source id) or *assumption*. The Supervisor
synthesises one verdict, then **pauses again** before writing anything to permanent memory.

> The promise is not "AI analyses your idea". It is: **every claim is traceable, every guess is
> labelled a guess, and the system is willing to say no.**

---

## Capstone metadata

| | |
|---|---|
| **Name** | Ghaidaa Alshareef, Rawan Alahmadi, Mayar Alhindi |
| **Programme** | Building Agentic AI Systems — SDAIA Academy |
| **Delivered by** | SDAIA Academy |
| **Trainer** | Mohammed Albeladi |
| **Cohort/session dates** | 23 August - 27 August 2026 |
| **SDAIA Academy on GitHub** | https://github.com/SDAIAAcademy |

---

## The problem

A first-time founder in Riyadh who wants to check an idea has three bad options, and
they fail in three different ways.

1. **General-purpose AI flatters.** Ask a chatbot "is my idea good?" and it finds something
   encouraging to say confident, fluent, and unfalsifiable. No stated confidence, no cited
   source, and no condition under which it would have said no.
2. **The knowledge gap is local, not general.** The model knows what a subscription revenue
   model is. It does not reliably know which licence a food-delivery business in Riyadh needs,
   which local players already occupy the space, or what Monsha'at requires. Answers calibrated
   to San Francisco, delivered to a market they do not describe.
3. **The output is not defensible.** Even when the answer is right, the founder cannot tell
   which sentence came from a source and which the model invented so it cannot be shown to an
   accelerator, an investor, or a co-founder.
---

## How it works

```
Founder's idea
     │
[0]  Intake                    → IdeaBrief (structured)
     │
[1]  Supervisor handoff        → calls transfer_to_market / _competitor / _business,
     │                           or request_clarification.  Every call is logged.
     │
[G0] ══ CLARIFICATION GATE ══  interrupt() only when the idea is too vague.
     │                          The founder answers → the idea is re-read and re-routed.
     │
[G1] ══ ANALYSIS PLAN GATE ══  interrupt() before any analyst spends a token.
     │                          Corrections here genuinely change the run.
     │
[2]  Market   │  Competitor   │  Business      ← parallel, no analyst sees another
     │  corpus │  web search  │  corpus
     │
[3]  Supervisor synthesis      → FinalReport
     │
[G2] ══ MEMORY GATE ══        interrupt() before the irreversible write to the Store.
     │
[4]  Persist → report + handoff ledger
```

### The agents

| Agent | Question it answers | Grounding |
|---|---|---|
| **Supervisor** | What is this idea, who should analyse it, what is the verdict? | The founder's stored profile and idea history |
| **Market** | Is there a real customer problem, and how big is the opportunity here? | Corpus (`search_corpus`) |
| **Competitor** | Who already serves this customer, and where is the gap? | Live web search — a static corpus can never name a competitor |
| **Business** | How does this make money, what licences does it need, what kills it? | Corpus — Monsha'at's own area of authority |

Analysts never communicate with each other. Every analyst reports back to the Supervisor.

### Routing is a handoff, not a keyword match

The Supervisor is given four tools `transfer_to_market`, `transfer_to_competitor`,
`transfer_to_business`, `request_clarification`, and **chooses which to call**. There is no
`if "market" in text` anywhere in this notebook. Every handoff is recorded in a ledger
(`show_handoffs`) with who called it, who accepted it, and the stated reason, so the routing
decision is inspectable rather than merely asserted.

Four outcomes are genuinely reachable: all three analysts, competitor only, business only, or
`request_clarification` which runs **no analyst at all**. That last branch is the one that
proves the Supervisor is *deciding* rather than dispatching.

### The `Claim` contract

Every factual statement any analyst emits is wrapped:

```python
Claim(statement=..., basis="evidence"|"assumption", source_id=..., confidence=0.0–1.0)
```

Three rules make the grounding rate impossible:

1. `basis="evidence"` with no `source_id` is downgraded to `assumption`.
2. A `source_id` that was **not actually offered to that analyst** is downgraded 
   `offered_ids()` extracts the exact ids from the analyst's own evidence block, so an invented
   URL cannot be laundered into an evidence claim.
3. The validators **coerce rather than raise**. A raising validator would blow up
   `with_structured_output` mid-run on a single model slip and destroy the whole analysis.

---

## Setup

### Run it in Google Colab

1. Open <https://colab.research.google.com> → **File → Upload notebook**
2. Add your keys in the Colab sidebar with **Notebook access** switched on
3. **Runtime → Run all**

### Keys

| Secret | Required | Free at |
|---|---|---|
| `GROQ_API_KEY` | **yes** | <https://console.groq.com> |
| `LANGSMITH_API_KEY` *or* `LANGCHAIN_API_KEY` | **yes** | <https://smith.langchain.com> |
| `SERPER_API_KEY` | optional | <https://serper.dev> — skip it and search falls back to DuckDuckGo |

Keys are resolved **Colab Secrets → environment variable → typed prompt**, so no key is ever
written into a cell and nothing sensitive can reach git. LangSmith renamed its key variable, so
both names are accepted; the key is verified with a single probe before tracing is enabled,
because a rejected key otherwise 403s on every run and buries the real output in warnings.

> **The variable that fails silently:** tracing uses `LANGCHAIN_TRACING_V2`, **not**
> `LANGSMITH_TRACING_V2`. The wrong name produces no trace *and* no error.

Embeddings run locally with `sentence-transformers/all-MiniLM-L6-v2` — no key, no cost.

### Model selection

Groq's free tier is **200,000 tokens per day, per model**. The notebook probes each candidate
model with a one-token request, skips any that is already spent, and deliberately puts the
heavy stages and the plumbing on **two different models** so they draw on separate daily
budgets. A spent model is out of budget, not broken and the notebook says which.

---

## Usage

```python
cfg = {"configurable": {"thread_id": "session-1"}}

# 1. submit  the run pauses at the first gate
r = analyze_idea.invoke({"idea": "A halal meal-prep delivery subscription for office "
                                 "workers in Riyadh. Full validation.",
                         "user_id": "mayar"}, cfg)
show_gate(r)          # prints the pending interrupt and the handoff ledger

# 2. approve, or correct the Supervisor's reading
r = analyze_idea.invoke(Command(resume={
        "approved": True,
        "corrections": "This is B2B corporate catering, not B2C.",
        "drop_workers": ["competitor"]}), cfg)

# 3. consent to what gets remembered
result = analyze_idea.invoke(Command(resume={"approved": True,
                                             "redact_memory_keys": []}), cfg)
show_report(result)
show_handoffs(result)
```

---

## Acceptance tests

| Test | What it proves | Pass condition |
|---|---|---|
| **T1** | The analysis is evidence-led | ≥ 60% of findings carry a verified `source_id` |
| **T2** | The system can say no | A deliberately weak idea returns `no_go` or `pivot` |
| **T3** | Real cross-thread long-term memory | A fact written in thread A is read in thread B |
| **T4** | It fails honestly | With the corpus emptied, the run completes in `degraded` mode and cites nothing |

**T2 and T4 are the two that separate this from a demo.** T2 proves the product has a spine;
T4 proves it fails honestly.

---

## Reliability four strategies, each mapped to a real failure

| Strategy | The failure it handles |
|---|---|
| **`RetryPolicy`** | Groq 429/503 under the concurrent load the fan-out itself creates. A real policy **object** no hand-written loops, no `time.sleep()` anywhere. |
| **Fallback to degraded mode** | The corpus is unavailable. Every claim becomes an assumption and confidence is capped **by deterministic code**, not by asking the model nicely. |
| **Graceful partial** | One analyst fails after its retries. Its section is marked unavailable and confidence lowered; the surviving analyses are still delivered. |
| **Retry on a different path** | `with_structured_output` forces `tool_choice`, so a model that answers in prose returns a hard 400 and its good text is discarded. `structured()` cascades tool-calling → `json_mode` → the other model. |

The fourth exists because the first three could not cover it, and that is the point:
`RetryPolicy` is for **transient** faults. A model that systematically refuses to emit a tool
call, or a daily quota that needs hours rather than seconds, is not transient retrying the
same model the same way cannot help. Distinguishing the two is what makes the error handling
real rather than decorative.

---

## Architecture decisions

**Agentic RAG, not 2-Step.** The founder's product description is never a usable search query,
analysts need several retrievals each informed by the last, and most importantly an analyst
needs the right to conclude *"the corpus does not cover this"*. A 2-Step chain always stuffs its
top-*k* into the prompt even when nothing relevant exists, which is exactly how ungrounded
confidence gets laundered into a report. Hybrid was rejected as unnecessary complexity for a
one-day MVP.

**The corpus has a search fallback.** Monsha'at is a JavaScript-heavy government site and a thin
scrape used to collapse the whole system into an ungrounded run. When the scrape falls below
threshold, the corpus is rebuilt from live search results for the same material real URLs,
real provenance. Only if *that* also fails does the system degrade. Per-URL character counts are
printed either way, so a silent scrape failure cannot pass unnoticed.

**Three gates, each guarding something irreversible.** The rubric asks for `interrupt()` before
an irreversible action; showing a report is reversible, so "approve the report" would not
qualify. Gate 0 asks for clarification instead of rejecting a vague idea. Gate 1 guards token
spend on a possibly-misread brief. Gate 2 guards a permanent memory write that shapes every
future verdict and asking is also the right privacy posture.

**Clarification is a round-trip, not a dead end.** A vague idea pauses, the founder answers, the
idea is re-read and re-routed, and only then does an analyst run. Bounded to one round, with a
clean exit if it is still too vague.

**Task boundaries speak plain dicts.** Every `@task` argument and return value is checkpointed,
and LangGraph warns on deserializing unregistered classes. The logic works in Pydantic models;
the task wrappers exchange `.model_dump()` dicts, keeping the checkpoint plain JSON which is
also what you want if you ever swap `InMemorySaver` for SQLite or Postgres.

---

## Project structure

```
.
├── ideas_analysts_capstone_v2.ipynb   # the deliverable — run top to bottom
├── PRD_v2.md                          # product requirements
├── README.md
├── requirements.txt
└── .gitignore
```

---

## Known limitations

1. **The corpus is narrow by design.** Monsha'at is authoritative on setup, licensing and
   planning, so the Business analyst is well grounded; it carries little market-sizing data, so
   the Market analyst leans on assumptions *visible* in T1's output, not hidden.
2. **Live scraping is fragile.** Mitigated by the search fallback above, but the corpus is
   rebuilt on every run and is not pinned, so two runs can be grounded in slightly different
   sources.
3. **In-memory persistence.** `InMemorySaver` and `InMemoryStore` reset when the Colab runtime
   restarts. Production would use `langgraph-checkpoint-sqlite` or Postgres same API surface.
4. **Bounded analyst loops.** A small number of tool-call rounds per analyst, a deliberate cost
   cap for a one-day MVP rather than an oversight.
5. **Verdict calibration is unmeasured.** T2 proves the system *can* say no on an obviously bad
   idea. It does not prove it says no at the *right* rate; that needs a labelled dataset.
