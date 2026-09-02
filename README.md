<div align="center">

# AgentReplay

**Replay a failed agent run, change one thing, and see what happens.**

[![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)](https://nextjs.org)
[![OpenTelemetry](https://img.shields.io/badge/OTel-GenAI%20semconv%201.42.0-blueviolet)](https://opentelemetry.io)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6?logo=typescript)](https://www.typescriptlang.org)
[![Tests](https://img.shields.io/badge/regression-3%20suites-brightgreen)](#testing)

</div>

<p align="center">
  <img src="./docs/assets/hero.png" alt="AgentReplay — automatic diagnosis, waterfall, and one-click Fork for ReAct agent runs" width="100%">
</p>

---

An agent run goes wrong: 14 steps, 40k tokens, a wrong answer delivered with confidence.

Logs show what happened. A dashboard shows when. Neither explains why, and neither lets you ask what the run would have done if one tool call had succeeded.

AgentReplay reads OpenTelemetry GenAI spans and builds three things on top of them: a structured timeline, an automatic diagnosis, and a fork. Pick a step, change one input, and run the rest again.

---

## Comparison with existing tools

| | Logs / `print()` | APM dashboards | AgentReplay |
|---|---|---|---|
| See the reasoning → tool → observation loop | ✗ flat lines | partial | ✓ agent-shaped waterfall |
| Names the actual failure mode | ✗ you read it | ✗ you read it | ✓ 8 automatic detectors |
| Re-run from a specific step | ✗ | ✗ | ✓ **Fork & Replay** |
| Test a counterfactual ("what if it hadn't timed out") | ✗ | ✗ | ✓ **override any tool result** |
| Prove a fix actually helped | ✗ | ✗ | ✓ side-by-side *Resolved / Persisted / Introduced* |
| Find systematically bad tools across all runs | ✗ | ~ | ✓ cross-run p50/p95 & error rates |
| Vendor-neutral input | ✗ | varies | ✓ **OpenTelemetry GenAI semconv** |

This is a debugger rather than a dashboard. A dashboard shows data and leaves the interpretation to you. This one names the problem, points at the spans it came from, and lets you run the experiment.

---

## Quickstart

Takes about a minute, and needs no API key.

```bash
git clone <repository-url> agent-replay && cd agent-replay
npm install

# Terminal 1: a fake LLM, so replay works without any setup
npm run mock:llm &

# Terminal 2: the debugger, wired to that fake LLM
REPLAY_LLM_BASE_URL=http://localhost:4010/v1 REPLAY_LLM_API_KEY=mock npm run dev

# Terminal 3: seed realistic failing traces
npm run mock
```

Open http://localhost:3000. The seeded traces contain real failure modes to work through: retry storms, error cascades, context growth.

> Replay works against a real provider too. Set `REPLAY_LLM_API_KEY` and skip the fake LLM. The seeded data is never sent anywhere.

---

## What it does

### 1 · A timeline shaped like the agent loop

The waterfall is not a flat list of spans. It shows the reasoning → tool → observation loop, a per-call token breakdown including reasoning tokens, and self time (duration minus children), so you can see where the wall-clock time actually went.

### 2 · Automatic diagnosis: 8 detectors

| Detector | Severity | Fires when |
|---|---|---|
| `retry-storm` | critical | 3+ consecutive failing calls to the same tool |
| `error-cascade` | critical | one failure cascades — names the **most specific** first failure, not the root span symptom |
| `reasoning-gap` | warning | a reasoning model reports no reasoning tokens (cost 5–20× understated) |
| `loop` | warning | the same call signature repeats consecutively |
| `context-growth` | warning | input tokens grew ≥4× across turns (quadratic cost) |
| `instrumentation` | critical/warning | deprecated or conflicting `gen_ai.*` attributes |
| `token-hotspot` | info | one call is ≥50% of all tokens |
| `latency-hotspot` | info | one span is ≥40% of wall-clock |

False alarms are treated as bugs. A debugger that reports problems that aren't there stops being read, so the detectors are deliberately conservative. Optional-reasoning models such as Claude 4.x, for example, are flagged only when the request actually turned thinking on, not because the model is capable of reasoning.

### 3 · Fork and replay

Select any step and click Fork. The conversation state at that point is rebuilt from the spans, and the agent is run forward again from there:

- Leave the tool results as recorded, and the result is a deterministic reproduction of the original behaviour. This confirms the trace can be trusted.
- Override one tool's output, and the result is a counterfactual: what the run does if `crm_lookup` returns successfully instead of 429.

Forking the seeded `support-agent` trace before its first failure:

| | Deterministic replay | Counterfactual (`crm_lookup` succeeds) |
|---|---|---|
| Steps | 3 | **2** |
| Tool calls | 2 (retry loop) | **1** |
| Tokens | 3750 | **2050** |
| Outcome | *"could not complete… escalating"* | **answered directly** |

That settles whether the retry storm was caused by the 429 or would have happened regardless. It is a measurement rather than an estimate.

### 4 · Comparison, to check a fix worked

`/compare?a=<original>&b=<replay>` diffs the two runs: metric deltas, span sequence, and a diagnosis diff split into three states:

- Resolved: present in the original, gone after the change.
- Persisted: still there, and unaffected by the change.
- Introduced: new in the replay, so a side effect of the change.

### 5 · Cross-run analytics

`/analytics` aggregates every ingested run over the `spans` index: per-tool and per-model call counts, error rates, p50/p95 latency, and token totals, sorted worst-first. This is how you find the tool that is always slow, rather than the one that was slow once.

---

## Architecture

```
  your agent
      │  OTLP/HTTP  (JSON or protobuf)
      ▼
  POST /api/v1/traces ──► normalize  ──►  buildTraceView  ──►  SQLite
                          (semconv         (tree, self-time,    (traces + spans
                           coalesce)        subtree rollups)     + raw_spans)
                                                │
              ┌─────────────────────────────────┼─────────────────────────┐
              ▼                                 ▼                         ▼
      /traces/[traceId]                    /analytics              insights (8 detectors)
       waterfall · diagnosis              cross-run p50/p95              │
              │                                                          │
              └──► Fork ──► replay engine ──► LLM ──► new spans ─────────┘
                            (self-instruments        │
                             via OTLP)               ▼
                                            /compare?a=..&b=..
```

The main design decision is that the replay engine does not depend on your agent's code. It rebuilds the conversation from spans, drives any OpenAI-compatible LLM, and exports its own spans back through the same OTLP endpoint, so a replay is stored as an ordinary trace. The lists, waterfall, insights and comparison views then work on it without any special handling.

---

## Connecting your own agent

Point any OpenTelemetry exporter at it. The receiver listens on the spec's conventional path `/v1/traces`, so the standard environment variable needs no path suffix:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:3000 \
OTEL_EXPORTER_OTLP_PROTOCOL=http/json \
OTEL_SERVICE_NAME=my-agent \
python my_agent.py
```

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:3000/v1/traces"))
)
```

`/api/v1/traces` is an equivalent alias if you prefer a namespaced path. Both accept `application/x-protobuf` (the SDK default) and `application/json`, along with `gzip` and `deflate` payloads. To check the endpoint is up, run `curl http://localhost:3000/v1/traces`.

Spans are read using the OpenTelemetry GenAI semantic conventions, pinned to semconv v1.42.0:

```
gen_ai.operation.name   = chat | execute_tool | invoke_agent | …
gen_ai.provider.name    = openai
gen_ai.request.model    = gpt-4.1
gen_ai.tool.name        = crm_lookup          # on execute_tool
gen_ai.usage.input_tokens / output_tokens / reasoning.output_tokens
error.type              = 429                 # standard OTel attribute
```

Already-instrumented SDKs (OpenAI Agents SDK, LangChain/LangGraph via OpenInference or OpenLLMetry, and similar) generally work without changes. Three generations of `gen_ai.*` attribute names are in circulation at the same time, so the normalizer coalesces them rather than summing them, and warns when it finds conflicting or deprecated names.

---

## Configuration

| Variable | Required | Notes |
|---|---|---|
| `REPLAY_LLM_API_KEY` | for replay | Not needed to explore traces |
| `REPLAY_LLM_BASE_URL` | no | Default `https://api.openai.com/v1`; any OpenAI-compatible endpoint |
| `REPLAY_LLM_MODEL` | no | Falls back to the model recorded on the span |

Everything runs locally. Traces are stored in `./data/traces.db` (SQLite).

---

## Project layout

```
src/
  app/            routes: / · /traces/[traceId] · /compare · /analytics · /api/v1/traces · /api/fork
  components/     waterfall, span inspector, fork panel
  lib/
    otlp/         wire types + hand-written protobuf decoder (rejects malformed payloads)
    genai/        semconv registry + normalizer (coalescing, data-quality warnings)
    trace/        tree building · ingest · insights · diff · analytics
    replay/       context reconstruction + replay engine
    db/           SQLite: traces · spans · raw_spans
scripts/
  mock-agent      seed synthetic failing traces
  mock-llm        zero-cost fake LLM for replay
  test-*.ts       regression suites (see below)
```

---

## Testing

Three regression suites, none of which need a running service:

```bash
npm run typecheck      # tsc --noEmit
npm run test:insights  # 8 failure modes: true positives AND false-alarm guards
npm run test:protobuf  # OTLP protobuf decoder round-trip + malformed-input rejection
npm run test:replay    # fork → replay → compare, end to end (needs dev + mock:llm)
```

`test:insights` feeds hand-built OTLP spans through the real pipeline (`normalizeSpan → buildTraceView → analyzeTrace`) and checks both directions: that the right detector fires, and that unrelated or healthy traces stay quiet. It includes a regression guard for an early false positive, where a non-reasoning model was flagged as a reasoning model.

A new detector is expected to ship with both a true-positive case and a case proving it stays quiet on healthy traces. Without the second one, the detector is not worth trusting.

---

## Known limitations

Listed in full, since a debugging tool you cannot trust is worse than none:

- OTel GenAI carries no tool parameter schema, so replayed tools get a permissive schema and the model infers the arguments. This is a gap in the specification rather than a missing feature here.
- Tools first used after the fork point have no recorded definition, so a replay rebuilds only what appeared earlier in the trace.
- Counterfactual fidelity depends on the substitute values you supply. The result shows whether the control flow changes, not that the substituted output was realistic.
- This is a local, single-process debugger (SQLite, no auth). It is not a production monitoring platform, and should be run on your own machine or an internal network.

---

## Built with

Five runtime dependencies (`next`, `react`, `react-dom`, `better-sqlite3`, `zod`) across roughly 8k lines of TypeScript, with no OpenTelemetry SDK, no LLM SDK and no ORM. The parts worth reading:

| File | Why it exists |
|---|---|
| `lib/otlp/protobuf.ts` | A protobuf wire decoder written by hand: varints, length-delimited fields, unknown-field skipping. It lets the receiver accept the OTel SDK's default `application/x-protobuf` without a proto runtime, and rejects malformed payloads instead of parsing them halfway. |
| `lib/genai/normalize.ts` | Three generations of `gen_ai.*` attribute names are live at once. This coalesces them by precedence and records data-quality warnings, rather than guessing or summing. |
| `lib/trace/insights.ts` | The 8 detectors, written with the false-alarm risk as the first constraint. |
| `lib/replay/context.ts` | Rebuilds an OpenAI-shaped message history from spans alone. This is what allows replay to work without importing any of your agent's code. |
| `lib/otlp/encode.ts` | The replay engine's own span exporter, so replays arrive through the same public ingest path as everything else. |

---

## Roadmap

- Surface systematically slow/erroring tools from analytics inside single-trace insights
- Group runs by `conversation.id` to debug multi-turn sessions
- Framework-specific instrumentation guides
