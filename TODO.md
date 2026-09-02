# Notes

Idea: agent runs fail and the logs never tell you why. Want to read OTel GenAI
spans and rebuild the run well enough to fork it at a step and re-run it.

Keep it vendor-neutral. Input is OTLP, not a proprietary SDK.

## Plan

- [x] OTLP/HTTP receiver — JSON first, protobuf after (OTel SDKs default to proto)
- [x] normalize gen_ai.* attributes; the names changed across spec versions
- [x] span tree with parent/child + self time
- [x] SQLite store, local only
- [x] mock agent that emits realistic failing runs, so there is data to look at
- [x] detectors for the failure modes worth naming
- [x] fork a run at step N, override one tool result, replay forward
- [ ] side-by-side compare of original vs replay
- [ ] cross-run analytics per tool/model
- [ ] README before this goes anywhere

## Open questions

- ~~Where does replay output go?~~ Replay exports through the same public
  /v1/traces path, so a replay is stored as an ordinary trace. No second path.
- Tools first used after the fork point have no recorded definition. Replay can
  only rebuild what appeared earlier in the trace. Document it, do not hide it.

## Decided

- Wrote the protobuf decoder by hand instead of pulling a proto runtime. It is
  varints + length-delimited fields + skipping unknown ones. Small enough.
- Keeping raw spans alongside normalized ones so nothing is lost on a schema
  guess. Prune them on a cap.
- Every detector needs a false-alarm test, not just a true positive. Got burned
  by a non-reasoning model being flagged as a reasoning model.
