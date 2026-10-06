# devin-langfuse-plugin

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](../LICENSE)

[English](../README.md)

ローカルの Devin session の transcript を [Langfuse](https://langfuse.com) に trace として送る [Devin CLI plugin](https://docs.devin.ai/cli/extensibility/plugins/overview)。既定では 1 turn = 1 observation で、ユーザー入力・最終応答・tool 呼び出し数・エラー・モデル別 token 使用量を保持する。

`Stop` hook が完了した turn を逐次 export し、`SessionEnd` が残りを flush するため、trace はほぼリアルタイムに見える。repo ごとの opt-in なので、対象外の repo には一切触れない。

## QuickStart

前提: サインイン済みの [Devin CLI](https://docs.devin.ai/cli)（`devin auth login`）と `PATH` 上の [uv](https://docs.astral.sh/uv/)。

```bash
# 1. plugin を install（personal manifest。全マシン + Devin Desktop に同期）
devin plugins install wwwyo/devin-langfuse-plugin#plugins/devin-langfuse

# 2. Langfuse の認証情報を env に（self-hosted なら LANGFUSE_BASE_URL も）
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_SECRET_KEY="sk-lf-..."

# 3. repo ごとに opt-in — これが無い repo では何も送られない
export DEVIN_TRACE_TO_LANGFUSE=true
```

`DEVIN_TRACE_TO_LANGFUSE` の gate は project dir ごとに評価される。repo 単位の env 機構（`mise.local.toml` の `[env]`、direnv、project 内だけで効く shell 設定など）で opt-in するのが想定の使い方。

opt-in した repo で Devin session を始めると、turn が session id 単位で Langfuse に届く。

## Features

- **増分・再開可能な export** — 各 turn は一度だけ送られる。checkpoint は配送が確認できてから確定し、retry は同じ deterministic observation ID で行われるため、再起動しても span は重複しない。
- **既定は compact** — `turn` mode は 1 turn 1 observation に `telemetry_summary`（generation/tool 件数・tool 名・bounded なエラー・model 別 usage）を載せる。`DEVIN_LANGFUSE_DETAIL=full` で generation 単位 + tool 入出力の詳細記録に切り替わる。
- **全面的に fail-open** — hook は常に exit 0 で detach して動き、`sessions.db` は read-only で開き、agent の挙動を止めたり書き換えたりしない。

cloud Devin session は対象外: plugin hook はローカル session（CLI と Devin Desktop）でのみ動く。

## License

[MIT license](../LICENSE)。
