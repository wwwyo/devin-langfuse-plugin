# devin-langfuse-plugin

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[日本語](docs/README.ja.md)

A [Devin CLI plugin](https://docs.devin.ai/cli/extensibility/plugins/overview) that ships your local Devin session transcripts to [Langfuse](https://langfuse.com) as traces — one observation per turn by default, with user input, the final response, tool-call counts, errors, and per-model token usage preserved.

Traces appear in near real time: the `Stop` hook exports each completed turn, and `SessionEnd` flushes the remainder. Per-repo opt-in keeps untraced repos completely untouched.

## QuickStart

Prerequisites: [Devin CLI](https://docs.devin.ai/cli) signed in (`devin auth login`), and [uv](https://docs.astral.sh/uv/) on your `PATH`.

```bash
# 1. Install the plugin (personal manifest, all your machines + Devin Desktop)
devin plugins install wwwyo/devin-langfuse-plugin#plugins/devin-langfuse

# 2. Export your Langfuse credentials (self-hosted: set LANGFUSE_BASE_URL too)
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_SECRET_KEY="sk-lf-..."

# 3. Opt in per repository — Devin traces nothing without this
export DEVIN_TRACE_TO_LANGFUSE=true
```

The `DEVIN_TRACE_TO_LANGFUSE` gate is evaluated per project directory, so a repo-local env mechanism (e.g. a `mise.local.toml` `[env]` entry, direnv, or your shell profile scoped to the project) is the intended way to opt a repo in.

Start a Devin session in the opted-in repo and watch turns land in Langfuse under the session id.

## Features

- **Incremental, resumable export** — each turn ships once; checkpoints commit only after confirmed delivery and retries reuse the same deterministic observation IDs, so restarts never duplicate spans.
- **Compact by default** — `turn` mode emits one observation per turn with a `telemetry_summary` (generation/tool counts, tool names, bounded errors, usage by model). Set `DEVIN_LANGFUSE_DETAIL=full` for per-generation spans with tool I/O.
- **Batching optional** — export is incremental per turn by default. Set `DEVIN_LANGFUSE_TIMING=session` to skip `Stop` events and flush the whole session once on `SessionEnd` (nothing is sent if the session never ends cleanly).
- **Fail-open everywhere** — the hook always exits 0, detaches its work, opens `sessions.db` read-only, and never blocks or rewrites agent behavior.

Cloud Devin sessions are out of scope: plugin hooks run in local sessions (CLI and Devin Desktop) only.

## License

Licensed under the [MIT license](LICENSE).
