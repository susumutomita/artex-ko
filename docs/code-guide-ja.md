# ソースから読む ARTEX

[日本語 README](../README.ja.md) · [利用案内](usage-ja.md)

対象は保管済み artex-ko のコミット `14b9f0c3b9a8db61d976086bb30d416e90846d14` です。ソースの静的読解で確認した範囲を説明します。実行・ビルド・依存インストール・攻撃の検証はしていません。原版との同一性も未確認です。

## まず押さえる構造

Go の HTTP サーバーがタスクと LLM エージェントを管理し、Next.js の画面が API と SSE で状態を読みます。planner は次の intent（意図）を作り、worker は一つの intent を担当します。保存された結果が次の planner ラウンドを起こす構成です。`engine/` という独立ディレクトリではなく、エンジンは `server/engine.go` にあります。

| ソース | 確認した責務 |
| --- | --- |
| [cmd/artex/main.go](../cmd/artex/main.go) | フラグ、更新 bootstrap、シグナル、Manager と HTTP サーバー生成、終了 |
| [config/config.go](../config/config.go) | 設定ファイル位置、DB DSN、skills の場所 |
| [server/manager.go](../server/manager.go) | PostgreSQL、記録プロキシ、タスク別 store・Guard の管理 |
| [server/server.go](../server/server.go) | API ハンドラー、タスク・会話・活動・設定等の HTTP 入口 |
| [server/auth.go](../server/auth.go) | パスワード関連処理、JWT の発行・検証と署名キー保存 |
| [server/engine.go](../server/engine.go) | planner/worker のループ、停止・再開・キャンセル・削除時の書込み障壁 |
| [agent/goals.go](../agent/goals.go) | 最終成果の目標分解と明示された操作制約・対象範囲の抽出 |
| [agent/mainagent.go](../agent/mainagent.go) | 人の会話を受け、ヒントや優先 intent で介入する入口 |
| [report/report.go](../report/report.go) | 保存済み情報から Markdown レポートを描画 |
| [traffic/traffic.go](../traffic/traffic.go) | HTTP 記録と SQLite 索引 |
| [enrich/enrich.go](../enrich/enrich.go) | 非 LLM の非同期 DNS・HTTP 資産補完 |
| [agent/planner.go](../agent/planner.go) | planner とグラフをつなぐ処理 |
| [agent/worker.go](../agent/worker.go) | 一つの intent の実行、作業場所、結果の書戻し、実行予算 |
| [agent/provider.go](../agent/provider.go) | LLM 設定の解決、provider の生成に関する構成 |
| [agent/assembly.go](../agent/assembly.go) | 基本ツールと skills/MCP の組立て、DB 側ツール解決、panic のエラー化 |
| [guard/guard.go](../guard/guard.go) | 実行前の承認・拒否・監査と実行後の分類 |
| [db/db.go](../db/db.go) | PostgreSQL 接続、埋込み schema の適用、初期設定の投入 |

## 起動から UI まで

`main()` は `run()` の返す終了コードをプロセスへ反映します。`run()` は自己更新 bootstrap を先に処理し、設定位置を確認して `server.NewManager()` と `server.New()` を生成し、`http.Server` を起動します。通常停止では終了コード0、更新による再起動要求では75を返します。[start.sh](../start.sh) はこのコードに応じて再起動します。

`Manager` は `db.Open()` で PostgreSQL を開き、記録プロキシが有効なら `traffic.Open()` を呼びます。タスクは共有資産 store とタスク固有の探索 store を持ちます。認証側は JWT キーをベースディレクトリに保存し、既定構成では閲覧可能な data ワークスペースから分離する設計です。`-data` の指定によってはこの分離を維持できないため、保存場所の関係を確認してください。

[web/src/i18n/config.ts](../web/src/i18n/config.ts) では対応 locale が `ko` と `zh`、既定 `ko`、ビルド時の指定が `NEXT_PUBLIC_LOCALE` です。画面全体のコンポーネント実装はこの案内では網羅していません。

## タスクとデータの流れ

1. HTTP API と Manager がタスクを管理し、タスクに探索 store と Guard を関連付けます。
2. Engine はタスクごとの planner と worker ループを管理します。LLM が未設定なら探索を行えません。
3. planner が状況を読み、intent を生成します。worker は intent を一つ担当します。
4. worker がツールを呼び、資産・事実・検出結果を store に書き戻します。実行ステップは activity として記録されます。
5. 変更通知が planner を再び起こします。SSE の活動配信は画面への更新経路です。

資産は複数タスクで共有する対象の情報、探索はタスクの進行と推論の記録です。[db/schema.sql](../db/schema.sql) に保存構造があり、`exploration_anchors` が探索ノードと資産を結びます。全テーブルとすべての SQL 操作をここで監査したわけではありません。

`goals` は最終的に欲しい成果と明示された制約・範囲の抽出を担い、`MainAgent` は人の介入窓口です。planner の通常ループとは別に、ヒントや優先 intent を渡せます。`report/report.go` の Markdown 描画は保存した情報を整形する経路です。

## LLM とツール呼び出し

`agent/provider.go` の `FromEnv()` は provider・モデル・キー・URL・LLM 用 proxy を環境変数から解決します。`ConfigFrom()` は UI 入力を設定へ変換します。対応形式は Anthropic、OpenAI Chat Completions、OpenAI Responses です。接続先やモデルの現時点の可用性は確認していません。

`agent/assembly.go` の `AugmentTools()` は基本ツールに可視 skills/MCP を追加し、`ToolResolve` が設定済みの有効性・説明・schema を反映します。MCP 用の遅延公開・unlock 情報も扱います。最終的にツールの panic を通常のツールエラーに変える wrapper を付けます。

worker は SDK の基本ツール群とグラフの書戻しツールを使う構成で、SDK 内部の実行実装そのものはこのチェックアウトに含まれず未読です。[go.mod](../go.mod) の norma 依存は v0.4.3 です。MCP サーバーや skills の中身・外部依存を安全と認定したものではありません。

## 安全境界の限界

`guard/guard.go` は `PreToolUse` / `PostToolUse` hook を登録します。実行前に監査情報を記録し、interceptor の判断を適用します。判断は `deny`、`allow`、`ask`。`ask` では承認を待ち、拒否・キャンセルなどで呼出しを遮断します。

重要なのは、Guard が OS・コンテナ・ネットワークの隔離ではないことです。ソース冒頭は以前の RoE（許可範囲）機構が削除されたと明記しています。interceptor がない場合、ツールが審査対象でない場合、ルール不一致かつ fallback judge が利用されない場合など、許可になる経路があります。破壊・持ち出し関連の制限は固定のコード上の下限ではなく、DB の設定可能なルールです。プロンプト上の制約も強制境界として扱えません。

記録 proxy や CA は通信記録のための機構で、すべての外向き通信を遮断・捕捉する保証ではありません。外部 SDK や各ツールのすべての通信経路は未監査です。API 認証も、モデルやツール実行を安全にする保証とは別です。

## 設定と保存

`config.json` は DB と skills の位置、LLM プロファイルは [db/config.go](../db/config.go) の DB 操作で管理します。プロファイルには API キーの保存フィールドもあるため、DB やバックアップを公開資料と混同しないでください。保存時の暗号化を保証した説明ではありません。

`db/db.go` は `schema.sql` を埋込み、起動時に接続・schema 適用・初期設定を行います。古いエントリポイントや provider のコメントには SQLite という記述が残っていますが、グラフ DB の現実の接続は PostgreSQL です。

`agent/worker.go` の `ensureRunDir()` はタスク用 `tasks/<taskID>` と worker 用 `i<intentID>` の作業場所を作ります。Manager は data 以下の traffic と transcripts に関係する処理も持ちます。`traffic/traffic.go` は記録用の SQLite 索引も持ちます。そのため全データが PostgreSQL だけに保存されるという理解は正確ではありません。通信記録は平文・未マスキングという前提がソースに記載されています。これらの完全な保存形式、削除・復元の全経路はこの案内の読解範囲外です。

## 読む順番と未読範囲

まず `cmd/artex/main.go` → `config/config.go` → `server/manager.go` の起動を読み、次に `server/engine.go` → `agent/planner.go` → `agent/worker.go` で一つの intent の循環を追うと理解しやすくなります。その後 `agent/assembly.go`・`agent/provider.go`・`guard/guard.go`、最後に `db/db.go` と schema を参照してください。大きいファイルは全行を説明したものではなく、ここで述べた経路を中心に読んでいます。

未読・未監査：norma の内部実装、全 MCP・skills のコード、全フロントエンド、全 API のアクセス制御、自己更新の検証手順全体、保存と復元の全経路。これらを推測して挙動や安全性の保証を追加していません。
