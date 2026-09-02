# AgentReplay

**Replay a failed agent run, change one thing, and see what happens.**

An agent run goes wrong: 14 steps, 40k tokens, a wrong answer delivered with confidence.

Logs show what happened. A dashboard shows when. Neither explains why, and neither lets you ask what the run would have done if one tool call had succeeded.

AgentReplay reads OpenTelemetry GenAI spans and builds three things on top of them: a structured timeline, an automatic diagnosis, and a fork. Pick a step, change one input, and run the rest again.

## Quickstart

Takes about a minute, and needs no API key.

```bash
npm install

# Terminal 1: a fake LLM, so replay works without any setup
npm run mock:llm &

# Terminal 2: the debugger, wired to that fake LLM
REPLAY_LLM_BASE_URL=http://localhost:4010/v1 REPLAY_LLM_API_KEY=mock npm run dev

# Terminal 3: seed realistic failing traces
npm run mock
```

Open http://localhost:3000. The seeded traces contain real failure modes to work through: retry storms, error cascades, context growth.

Replay works against a real provider too. Set `REPLAY_LLM_API_KEY` and skip the fake LLM.

## What it does

**A timeline shaped like the agent loop.** The waterfall is not a flat list of spans. It shows the reasoning → tool → observation loop, a per-call token breakdown, and self time, so you can see where the wall-clock time actually went.

**Automatic diagnosis.** Eight detectors run over every trace: retry storms, error cascades, reasoning gaps, loops, context growth, instrumentation gaps, token hotspots, latency hotspots. Each one names the failure mode instead of leaving you to read it out of the spans.

**Fork and replay.** Pick a step, optionally override one tool result, and re-run the rest. The conversation is rebuilt from the spans alone, so none of your agent's code is needed. The replay exports through the same ingest path, which means it is stored as an ordinary trace.

**Comparison.** Put the original and the replay side by side. Each finding comes back as resolved, persisted, or introduced.

## Ingest

Point any OpenTelemetry exporter at the receiver:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:3000
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Both `/v1/traces` and `/api/v1/traces` accept OTLP/HTTP in JSON and protobuf, gzip or deflate. Spans are read against the GenAI semantic conventions (semconv 1.42.0).

## Project layout

```
src/
  app/            routes and pages
  components/     waterfall, span inspector, fork panel
  lib/
    otlp/         wire types + hand-written protobuf decoder
    genai/        semconv registry + normalizer
    trace/        tree building · ingest · insights · diff · analytics
    replay/       context reconstruction + replay engine
    db/           SQLite: traces · spans · raw_spans
scripts/
  mock-agent      seed synthetic failing traces
  mock-llm        fake LLM for replay
  test-*.ts       regression suites
```

## Testing

```bash
npm run typecheck
npm run test:insights
npm run test:protobuf
npm run test:replay
```

`test:insights` feeds hand-built OTLP spans through the real pipeline and checks both directions: that the right detector fires, and that healthy traces stay quiet.
