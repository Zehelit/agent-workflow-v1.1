<a name="repeatable-lifecycle-check"></a>

# 再現できるライフサイクルの確認

`tooling/run-lifecycle-check.py` は、実際に観測した目標、使用量、子エージェント一覧を固定候補プログラムで処理します。ホストから観測を得るのはメインエージェントです。セッションと入力は Git の外に非公開で保管し、書き込み担当を一つにします。

<a name="prepare-before-measuring"></a>

## 測定前に準備する

candidate をビルドし、受け入れた外部固定値を保存します。`prepare` はパッケージを検証してセッションを記録するだけで、candidate や監視は起動しません。実際の初回観測を取る前に準備を終えます。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` は受け入れ済みのビルド記録から取得します。既存プロジェクトと新しいセッションファイルを指定します。パッケージ変更後に以前のセッションを別のハッシュに結び直さないでください。

<a name="host-input-object"></a>

## ホストから渡す入力

`run`、`observe`、`resume`、`close`、`timed-test` は `--input` で JSON オブジェクトを受け取ります。

| フィールド | 内容 |
| --- | --- |
| `activation` | 初回の `run` のみ。`bind-active-goal` の型付き入力で、`native_binding`、一致する `app_goal_observation`、目標、解釈、手順の確認、実際に観測したモデル・推論レベル、枠の上限、一覧と観測時刻を含みます。 |
| `app_goal_observation` | 最新の実 `get_goal` の目標・会話 ID、目標文、作成時刻と状態、`source_kind: get_goal`、実際の出典参照、`observed_at`。取得できた実 `updated_at` も含めます。 |
| `usage` | [ネイティブ使用量の記録](../skills/agent-workflow-v1-1/references/native-usage.md)の `source`、`sourceId`、`observedAt`、`result`。必要な枠だけを保存します。 |
| `inventory` | 実際の子エージェントの ID と現在の状態。子がいない場合のみ `[]`。 |
| `checkpoint_files` | メインが書いたものを含む、復元したい変更・追加・削除のプロジェクト相対パス。コアは担当ファイルも追加します。 |
| `writers_quiescent` | 書き込み担当が全員停止したと観測した場合に限り `true`。このツールは観測を作れません。 |
| `next_action` | 復旧または通常終了後の具体的な次の手順。 |
| `weekly_exception` | 初回 `run` だけの任意項目。ユーザーの実際の `exact_quote` と有限の Unix 期限 `valid_until`。既定は例外なしです。 |
| `resume` | 任意の完全な candidate の再開入力。なければ記録済み期限切れにのみ `usage_recovery: true` を試します。上限やユーザー停止には別の要件があります。 |

ネイティブ API に独立した目標 ID がないときは `threadId:createdAt` など一貫した派生識別子を使い、派生値と明記します。型の詳細は[運用](../skills/agent-workflow-v1-1/references/operations.md)と[コマンド](../skills/agent-workflow-v1-1/references/commands.md)を参照してください。

<a name="run-refresh-and-close"></a>

## 実行、更新、正常終了

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

ネイティブ呼び出しが戻った時点の実観測時刻を記録します。ツールは `status.at == usage.observedAt`、所有プロセス、エポック、ハートビート、サンプル、リースを確認します。`observe` はリースも更新します。`refresh_by` は早めに更新する目安で、実際の期限は変えません。

`CHECKPOINT_SOON` では完了した作業のチェックポイントを検証し、`checkpoint_files` を最新に保ちます。通常の `close` には最新の監視、検証済みスナップショット、完了、終了したプロセスの観測が必要です。通常操作は SAVE を人工的に発生させません。

最初の `run` に `weekly_exception` がある場合、ツールは有限ポリシーを監視に設定して `NEED_POLICY_OBSERVATION` を返します。まだ有効化されません。実際の目標と使用量を再観測し、同じセッションでやり直してください。以前の記録の時刻は維持します。

<a name="halt-and-resume-after-expiry"></a>

## 期限切れ後の停止と再開

監視が SAVE を要求すると、`observe` は stop、snapshot、halt を行い、`NEED_POST_HALT_OBSERVATION` を返します。停止後に目標、使用量、一覧を改めて取得します。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

コアは監視で観測された理由を使用します。技術復旧には、許可済みの同じ有効な目標、変更されていないスナップショットとファイル、停止後の新しい観測、所有プロセスの終了が必要です。枠の枯渇、早期停止、ユーザーによる停止には別の続行条件があります。ツールはネイティブ目標を paused、blocked、active に変更しません。

初回起動の失敗時、セッションが candidate の試行ハッシュを保存します。定められた再試行条件だけで、新たな証拠を使って同じ `run` を実行します。変更されたセッションは承認済み証拠になりません。完了済み、または結果不明の試行は自動で再開しません。

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## 指定された場合だけ時刻を診断する

イベントより前に正常な監視を準備します。`timed-test` は明示的に要求された診断用です。イベント JSON の `event_id`、`goal_id`、`thread_id`、`source_id`、`event_at`、`observed_at` は実際のネイティブ観測に基づきます。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

遅れて得たイベントは `INVALID_TIMING` になります。`SAVE_REQUEST_ON_TIME` は SAVE の要求時刻を示します。ネイティブ停止には別の証拠が必要なため `native_pause_verified: false` と `native_paused_at: null` を残します。実環境の二回確認では、同じ目標・実行の active、halted、許可された続行後の active を観測します。合成の時計はロジックの検証に限られます。
