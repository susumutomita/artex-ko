# 起動・設定・通常利用の日本語案内

[日本語 README](../README.ja.md) · [韓国語原文](../README.md) · [英語原文](../README.en.md)

既存文書の起動・設定部分を要約翻訳しました。所有するローカル隔離環境での学習・研究という前提を維持してください。本作業ではインストール・ビルド・起動・モデル呼び出しを実行していません。

## 起動経路

既存 Compose は PostgreSQL と ARTEX を起動し、`skills` と `data` をホストに保存する構成です。[docker-compose.yml](../docker-compose.yml) の `autumn27/artex` イメージは、原文によれば2026-10-09時点で取得不能でした。クイックスタートを現在動作する保証のある手順と受け取らないでください。

原文は install.sh、リリースのバイナリ、ソースからの単一バイナリのビルドを案内しています。install.sh の Docker 経路も同じイメージに依存します。ソース経路は Next.js の静的ビルドを `server/webui/dist` に同期し、Go の `embedui` タグ付きバイナリに埋め込む構成です。[既存の具体的手順](../README.en.md#other-installation-methods) を保持しています。原文の「正常動作を確認」は原文著者の報告で、日本語化で追試した結果ではありません。

原文の注意として、`npm ci` には開発依存も必要です。`NODE_ENV=production` により開発依存が省かれると静的ビルドが失敗することがあります。再ビルド時の静的出力の同期にも注意が必要です。

## 起動ラッパーと待受

原文は `start.sh`（Windows は `start.bat`）を利用するよう案内します。[start.sh](../start.sh) は終了コード0で停止、75で即時再起動、それ以外で最大60秒までのバックオフ再起動を行います。更新の検証・交換は Go の `selfupdate` 側です。

[cmd/artex/main.go](../cmd/artex/main.go) のフラグ：

| 識別子 | ソース上の既定値・意味 |
| --- | --- |
| `-addr` | `:8787`：HTTP API と UI の待受 |
| `-proxy` | `127.0.0.1:8788`：記録プロキシ。空文字は無効化指定 |
| `-data` | 実行ファイルに隣接する `data`：実行時ファイルの場所 |

既存 README の `-proxy :8788` は明示指定例で、現在のソースの既定値とは異なります。HTTP の既定待受は loopback 限定ではありません。隔離環境外への露出を防ぐ構成を確認してください。

## DB と skills

[config.example.json](../config.example.json) のキーを維持して `config.json` を用意する設計です。ファイルは `database` と `skill_dir` を持ち、LLM 設定をここに追加する設計ではありません。

[config/config.go](../config/config.go) で確認した優先順位：

1. 設定ファイル：`ARTEX_CONFIG`、作業ディレクトリの `config.json`、実行ファイルに隣接する `config.json`。
2. DB：`ARTEX_PG_DSN`、ファイルの `database.dsn`、ファイルの `database` 個別フィールド。DSN が非空なら個別フィールドは使いません。設定がなければエラーです。
3. skills：`ARTEX_SKILL_DIR`、ファイルの `skill_dir`、実行ファイルに隣接する `skills`。相対パスは作業ディレクトリ基準です。

`database` のフィールドは `host`、`port`、`user`、`password`、`dbname`、`sslmode`。設定例のポートは5433、Go の未指定時は5432です。接続先に合わせてください。実際のパスワードや DSN を文書・Git に含めないでください。

## LLM と出力言語

キーは `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`、任意設定は `ARTEX_LLM_PROVIDER`、`ARTEX_LLM_MODEL`、`ARTEX_LLM_BASE_URL`、`ARTEX_LLM_PROXY` です。UI の LLM 設定でもプロファイルを管理します。ソースは `anthropic`、`openai`、`openai-responses` を扱います。未指定 provider はキーから推定し、両方あれば Anthropic を先に選びます。

キー、プロンプト、会話に含まれる情報はモデル接続先への送信に関係します。秘密設定値・社内ソースを公開文書に貼り付けないでください。

韓国語出力は `agent/prompt.go` の `langDirective()` などの指示で誘導されます。モデル・役割・文脈により他言語が混ざることがあります。日本語文書の追加は出力言語の変更を意味しません。原文のモデル比較やトークン上限は当時の報告です。現在の品質・サービス上限は本作業では確認していません。`max_tokens` を使用する接続先の制限に合わせてください。

## 通常利用

以下は既存文書の画面機能の説明です。本作業では実際のタスクを作成・実行していません。

- 初回は `/setup` で管理者パスワードを設定し、その後ログインします。
- LLM 設定で接続先を管理し、システム設定で worker 同時数を確認します。原文の既定数は3です。
- タスク画面では目標・状態・活動・探索グラフ・資産・検出結果・レポートを確認します。会話で人が補足する機能もあります。
- 検出結果は深刻度や状態などで集計し、CSV 出力を提供します。
- ワークスペースとトラフィック記録には実行時ファイルや通信内容が含まれ得ます。公開成果物と混同しないでください。
- [サイド質問 `/btw`](../sidequestion/README.ja.md) は本文から分離した質問です。

## リバースプロキシと SSE

原文は UI・API・SSE が同じバックエンドから提供されるため、通常 `NEXT_PUBLIC_SSE_BASE` の別指定は不要と説明しています。HTTPS リバースプロキシでは SSE のバッファリングを無効にしないとイベントが遅延します。[既存中国語の設定例](../README.zh.md#反向代理部署https--只开放-443) を保持しています。外部公開の推奨ではなく、ここでは追試していません。
