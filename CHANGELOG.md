# Changelog

## v1.1.0 — 2026-06-14
Optional API keys (GitHub, libraries.io, Stack Overflow) are now surfaced **once** with signup URLs and graceful degradation — no longer silently skipped.

## v1.0.0 — 2026-06-13
- Initial public release.
- Parallel-subagent project discovery: fans out research across GitHub,
  Libraries.io, and the Claude/AI ecosystem, then synthesizes a recommendation.
- Optional credential files (`github-token`, `libraries-io-key`,
  `stackoverflow-key`) documented in SETUP.md for authenticated, higher-rate searches.
