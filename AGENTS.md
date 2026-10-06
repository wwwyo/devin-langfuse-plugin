# devin-langfuse-plugin

A [Devin CLI plugin](https://docs.devin.ai/cli/extensibility/plugins/overview) that ships local Devin session transcripts to Langfuse as OTLP traces, via `Stop` (per-turn increment) and `SessionEnd` (final flush) hooks.

## Directory structure

```
devin-langfuse-plugin/
├── plugins/
│   └── devin-langfuse/        # the plugin root — installed via `devin plugins install <repo>#plugins/devin-langfuse`
│       ├── .devin-plugin/plugin.json
│       ├── hooks.json              # Stop + SessionEnd registrations
│       └── hooks/                  # hook wrapper, exporter, vendored emit library, tests
├── docs/                           # shared docs (tracked); PRDs live in docs/prd/<topic>/
└── .github/                        # CI, dependabot, pullfrog.config.sh
```

Everything under `plugins/devin-langfuse/` ships to machines where the plugin is installed. Repo-only files (CI, dev docs) stay outside it. Never add an `AGENTS.md` or `rules/` inside the plugin root — they are injected as always-on rules into every session that installs the plugin.

## Setup

Tools are managed with mise.

```bash
mise install
```

```bash
# regression suite: real pinned SDK + a localhost OTLP collector; no cloud access
uv run --script plugins/devin-langfuse/hooks/test_langfuse_export.py

# lint
shellcheck plugins/devin-langfuse/hooks/*.sh
jq empty plugins/devin-langfuse/.devin-plugin/plugin.json plugins/devin-langfuse/hooks.json
```

## Tech stack

- POSIX sh hook wrapper + Python single-file scripts executed via `uv run --script` (PEP 723 inline metadata).
- Python deps (`langfuse`, `requests`) are exact-pinned inside each script's `# /// script` block — Dependabot does not scan PEP 723 metadata, so bump pins manually (7-day cooldown policy).
- `plugins/devin-langfuse/hooks/langfuse_hook.py` is a vendored copy of the Langfuse Claude Code hook with documented divergences (deterministic source IDs, compact `turn` mode, per-source labels); it also depends on Langfuse SDK 4.x internals — bump SDK pin and re-verify together.

## Behavior invariants (do not regress)

- Repo opt-in only: `DEVIN_TRACE_TO_LANGFUSE=true` (repo-local `mise.local.toml` `[env]`). An opted-out repo exits before touching uv or the network.
- Hook always exits 0 and detaches the exporter via `nohup`; `sessions.db` is opened read-only.
- Default detail is `turn` (one observation per turn, `metadata.telemetry_summary` version=1); `DEVIN_LANGFUSE_DETAIL=full` for per-generation detail. Switching modes must not replay history (fingerprints exclude the mode).
- Default timing is per-turn (`Stop` incremental + `SessionEnd` final flush); `DEVIN_LANGFUSE_TIMING=session` skips `Stop` in the wrapper only. A payload whose `hook_event_name` is missing or unrecognized must not be skipped — degrading to incremental is safer than losing telemetry entirely.
- Checkpoints are source message-ID fingerprints keyed under `devin::<session_id>` in `~/.local/state/langfuse-export/state.json`; a turn's checkpoint commits only after confirmed delivery (HTTP error, OTLP partial rejection, or flush failure leaves it pending for retry under the same deterministic IDs).
- Never emit raw tool bodies or intermediate assistant text in `turn` mode; keep the `telemetry_summary` contract (`version=1` fields) that downstream session-eval reads.

## Skills

- For repository-local learned notes, check `.agents/skills/<topic>/` when present.

## Pullfrog

Server-side review settings are recorded in `.github/pullfrog.config.sh` (SSOT; reapply with `bash .github/pullfrog.config.sh`). `.github/workflows/pullfrog.yml` is Pullfrog-managed — do not edit.
