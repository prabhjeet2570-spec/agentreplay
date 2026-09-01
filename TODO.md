# Notes

Idea: agent runs fail and the logs never tell you why. Want to read OTel GenAI
spans and rebuild the run well enough to fork it at a step and re-run it.

Keep it vendor-neutral. Input is OTLP, not a proprietary SDK.

## Plan

- [x] OTLP/HTTP receiver — JSON first, protobuf after (OTel SDKs default to proto)
- [x] normalize gen_ai.* attributes; the names changed across spec versions
- [x] span tree with parent/child + self time
- [x] SQLite store, local only
- [ ] mock agent that emits realistic failing runs, so there is data to look at
- [ ] detectors for the failure modes worth naming
- [ ] fork a run at step N, override one tool result, replay forward
- [ ] side-by-side compare of original vs replay
- [ ] cross-run analytics per tool/model

## Open questions

- Where does replay output go? Probably back through /v1/traces so a replay is
  just another trace. Avoids a second storage path.
- How much of the conversation can be rebuilt from spans alone?

## Decided

- Wrote the protobuf decoder by hand instead of pulling a proto runtime. It is
  varints + length-delimited fields + skipping unknown ones. Small enough.
- Keeping raw spans alongside normalized ones so nothing is lost on a schema
  guess. Prune them on a cap.
