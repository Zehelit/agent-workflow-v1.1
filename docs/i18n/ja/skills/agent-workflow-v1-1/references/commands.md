<a name="command-contract-a02"></a>

# コマンドの仕様（A02）

標準ライブラリで実装した補助プログラムは、`Workflow.execute` と `Workflow.apply` を通るコマンドを検証します。ネイティブツールを横取りしたり、ツールの実行記録やユーザーメッセージの真正性を確認したり、プロセスを停止したり、実際に使われたモデルを独自に観測したりはしません。これらはメイン側が行い、結果を証跡として渡します。A26 の権限確認、チェックポイント、所有関係、容量の制約は引き続き適用されます。

状態の `schema_version: 2` は整数です。バージョンの指定がない状態、旧版、将来の版、形式が壊れた状態は、変更前に拒否します。旧状態の移行・取り込みツールはありません。現在の型付きの権限情報で新しい AGWF 実行を始めてください。チェックポイントの取得と復元は利用できます。過去の証跡だけで作業の権限は生まれません。障害を注入するテストには使い捨てのフィクスチャのコピーだけを使います。

<a name="json-and-error-policy"></a>

## JSON とエラーの方針

状態ファイル、`--input` のファイル、標準入力、チェックポイントのメタデータ、インストーラーの所有関係を示す JSON は、共通の厳密なパーサーで読み取ります。どの階層でもキーの重複を拒否し、値が同じ場合も例外にしません。`NaN`、`Infinity`、`-Infinity` および有限に見えても無限大にオーバーフローする数値リテラル（`1e999` など）は拒否します。入力ファイルは BOM の有無を問わず UTF-8 を受け付けます。コマンドの JSON ルートはオブジェクトである必要があります。空の標準入力は `{}` となり、コマンドの仕様が許す場合だけ成功します。`write_json` は、ディレクトリや一時ファイルを作成する前に `allow_nan=False` で検証し、シリアライズします。

ランタイムのペイロードは、キーが文字列の JSON オブジェクトです。リストはリスト、整数と浮動小数点数は数値として扱い、真偽値と区別します。指定されたフラグはすべて厳密に Python の `bool` 型（JSON では `true`／`false`）でなければなりません。文字列、0/1、null、リスト、オブジェクトで代用できません。必須フラグの省略は拒否します。任意フラグの既定値が false になるのは省略時に限られ、null は不正です。数値は有限でなければなりません。件数は 1 以上の整数、タイムスタンプは 0 以上 sys.float_info.max 以下の整数か浮動小数点数、パーセントは 0 から 100 までです。タイムスタンプの上限により、時刻の演算でオーバーフローする巨大な整数を拒否します。さらに、各コマンドで定めた時間の条件を満たす必要があります。容量の上限は追加しません。割り当ては従来どおりメインを枠に含め、子エージェント数の上限も適用します。

ペイロードの検証は、ロックやディレクトリを作成する前、かつコマンドを実行する前に行います。`apply` は非公開の JSON のコピーで処理し、条件を通過した後に反映します。拒否されたコマンドは、以下で明示する永続的な INPUT_INVALIDATED の記録と contract_invalid のフラグを除き、状態ファイルのバイト列と SHA-256 を保持します。初回コマンドが拒否された場合は状態を作成しません。スナップショットが失敗すると、従来どおり、確定前の所有チェックポイント用ディレクトリが残る場合があります。入力形式は正しいものの使用量を取得できない場合の状態遷移は、意図された動作です。後述します。

補助プログラムはエラー時に終了コード 1 を返し、標準エラー出力に次のオブジェクトを書きます。

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| コード | 意味 | 対象フィールド |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | 不明なエンジンコマンド | `$.command` |
| `FIELD_REQUIRED` | 必須の入力フィールドまたはユーティリティの必須オプションがない | 欠けているフィールド |
| `FIELD_UNKNOWN` | 不明な入力フィールド、またはそのコマンドで使えない CLI オプション | 該当するフィールド |
| `FIELD_TYPE` | 呼び出し側が指定した型が不正 | 該当するフィールドまたはリストの位置 |
| `FIELD_VALUE` | 識別子、必須の文字列・リスト、数値の範囲、実行一覧の状態値が不正 | 該当するフィールドまたはリストの位置 |
| `JSON_DUPLICATE_KEY` | JSON メンバーが重複 | 入れ子も含め、重複したメンバー |
| `JSON_NONFINITE` | 生のメタデータを含むいずれかの場所に有限でない数値またはトークンがある | 該当するフィールドまたはリストの位置 |
| `JSON_SYNTAX` | JSON の構文またはエンコーディングが不正 | `$` |
| `JSON_TYPE` | 実行時の値が JSON 型でないか、オブジェクトのキーが文字列でない | 該当する値またはオブジェクト |
| `COMMAND_REFERENCE` | 参照するタスクまたは保存済み必須フィールドが存在しない | 検索の対象は `$.id`。欠落キーはエラー本文に出る |
| `COMMAND_CONSTRAINT` | 既存の権限、状態、所有関係、レビュー、計測の鮮度に関する制約 | 特定できる場合はコマンドの該当フィールド。状態全体の制約は `$.command` |
| `STATE_VERSION`／`STATE_TYPE` | 未対応の状態バージョン、または状態のルートがオブジェクトでない | `$.schema_version`／`$` |
| `IO_ERROR`／`INVALID_VALUE` | ファイルシステムの障害、またはその他の値の不正 | `$` |

パスは単純なキーにドット表記、それ以外のキーにブラケット表記、リストに `[index]` を用います。通常の argparse のエラー（不明な CLI スイッチやコマンド、スイッチの引数不足）は、argparse の標準エラー出力と終了コード 2 をそのまま使います。認識できるものの当該コマンドで無効なオプションは、上の構造化された `FIELD_UNKNOWN` エラーになります。どちらも `assert` に依存しないため、`python -O` でも検証は働きます。

<a name="payload-types"></a>

## ペイロードの型

`text` は、検証のために前後の空白を除いた後も空でない文字列です。元の内容は保持します。`string` は空でも構いません。`task_id` は `[a-z][a-z0-9_]{0,63}` に一致する識別子です。`strings` は空でない文字列のリストで、リスト自体は空でも構いません。`criteria` は最低 1 項目を必要とします。`task_ids` はタスク ID のリストです。`timestamp`、`count`、`percent` は上記の数値規則に従います。明示的な null を許すのは `?` が付いた型だけです。

以下の各コマンドと入れ子の構造化オブジェクトは、必須・任意の一覧にないフィールドを拒否します。プロバイダーの生データを保持する例外が二つあります。観測記録の内容と、生の `rateLimits` および各枠の計測値以外のメタデータは、内容を解釈しない JSON として扱います。この場合でも重複キー、有限でない値、JSON で表せない値は拒否します。未知のメタデータをコマンドのフラグや有効な計測の証拠とは解釈しません。

| 入れ子の型 | 必須フィールド | 任意フィールド・制約 |
| --- | --- | --- |
| `authorization` | `event_id`、`exact_quote`、`kind`、`source_kind`、`source_id`、`goal_id`、`thread_id`、`run_id` は text。`scope` は scope。`observed_at`、`source_created_at` は timestamp。`original_turn_id`、`observed_turn_id` は text | 追加フィールドは不可。A26 の許可された種別、ユーザーから直接得た出所、範囲の一致、再送と経過時間の制約を適用する |
| `scope` | `kind`、`id`、`objective` は text | `kind` は A26 の制約に従う objective または report |
| `source` | `source_kind`、`source_id`、`exact_quote`、`original_turn_id` は text。`source_created_at` は timestamp | 渡された権限情報から抽出した出所フィールドと一致する必要がある |
| `native_binding` | `binding_id`、`goal_id`、`thread_id`、`run_id`、`objective` は text。`scope` は scope。`goal_created_at`、`observed_at` は timestamp | N02 の初回ひも付け専用。範囲の種別は objective とし、目的と厳密に一致させる。ゴールの作成は観測より前。人を出所とするフィールドは持たない |
| `app_goal` | `goal_id`、`thread_id`、`status`、`source_kind`、`source_id` は text。`observed_at` は timestamp | `objective`（text）と `goal_created_at`（timestamp）は任意。ただしネイティブの権限根拠を使う場合は毎回の更新も含めて必須で、ひも付けと一致させる。A26 では get_goal を出所とし、識別情報が一致し、観測から 60 秒以内で、状態が用途に応じてアクティブである必要がある |
| `inventory` | `id` と `status` が text のオブジェクトのリスト | 項目ごとに追加フィールドは不可。ID は一意。状態は running、interrupting、unknown、idle、stopped、done。観測から 0〜60 秒 |
| `models` | 空でないモデル名を `strings` 型の推論強度リストに対応付けるオブジェクト | 空のオブジェクトやリストも可。prepare には観測で確認した正確なモデルと推論強度が必要 |
| `arguments` | 構造上の必須フィールドなし | 使用できるのは `task_name`（task_id）、`fork_turns`（text）、`message`（string）、`model` と `reasoning_effort`（text）のみ。record は準備済み引数との完全一致も求めるため、フィールドが欠けていれば割り当てを拒否する |
| `receipt` | 空でないオブジェクト、リスト、または文字列 | 実際のツール結果を、内容を解釈しない JSON として扱う。真正性は保証しない |
| `rate_limits?` | オブジェクトまたは null | primary と secondary は任意の window?。追加のプロバイダーメタデータは内容を解釈しない JSON |
| `window?` | オブジェクトまたは null | `usedPercent`（percent?）、`windowDurationMins`（count?）、`resetsAt`（timestamp?）は任意。それ以外のメタデータは内容を解釈しない JSON |

<a name="engine-commands"></a>

## エンジンのコマンド

次の必須・任意フィールドの一覧がすべてです。任意フィールドの記載がないコマンドには必須フィールドだけを渡します。入力項目のないコマンドには `{}` を渡します。

| コマンド | 必須フィールド | 任意フィールドと既定値 |
| --- | --- | --- |
| `activate` | `objective`（text）、`authorization`（authorization）、`interpretation`（string）、`source_message`（source）、`app_goal_observation`（app_goal）、`models`（models）、`slot_limit`（count）、`inventory`（inventory）、`observed_at`（timestamp） | `session_limit`（count?）。省略または null なら、開いているセッション数に追加上限を設けない |
| `bind-active-goal` | `objective`（text）、`native_binding`（native_binding）、`interpretation` と `procedural_check`（text）、`app_goal_observation`（app_goal）、`models`（models）、`slot_limit`（count）、`inventory`（inventory）、`observed_at`（timestamp） | `session_limit`（count?）。schema2 の非アクティブな初期状態で、タスクが空、他の履歴やフィールドが一切ない場合に限る |
| `observe-authorization` | `authorization`（authorization）、`interpretation`（string）、`source_message`（source） | `app_goal_observation`（app_goal）。受け入れ対象のフィールドを渡せるが、保存済みのゴール観測は更新しない |
| `app-goal` | `app_goal_observation`（app_goal） | なし |
| `usage` | なし | `observed_at`（timestamp）、`rateLimits`（rate_limits?）。省略は計測不能を表す |
| `usage-due` | なし | `substantial_work`（bool、既定値 false） |
| `inventory` | `inventory`（inventory）、`observed_at`（timestamp） | なし |
| `task` | `id`（task_id）、`role`（text）、`objective`（text）、`criteria`（criteria） | `file_inputs`（file_inputs、[]）、`context_refs`（strings、[]）、`review_targets`（review_targets、[]）、`revision`（text、省略時に UUID 生成）、`writes`（strings、[]）、`depends_on`（task_ids、[]）、`independent_review`（bool、false） |
| `prepare` | `id`（task_id） | `effort`（text、省略時は役割の既定値）、N06 の `current_context`（context_pin） |
| `record` | `id`（task_id）、`submitted_arguments`（arguments）、`session_id`（text）、`receipt`（receipt） | `effective_model`、`effective_effort`（text?）。省略または null は未検証 |
| `dispatch_failed` | `id`（task_id）、`evidence`（text） | `certain_not_started`（bool、false）。省略または false なら起動の有無は不明 |
| `reconcile` | `id`（task_id）、`evidence`（text）、`no_owned_work_running`（bool） | true と、稼働中の子エージェントがいないことを示す新しい実行一覧が必要 |
| `finish` | `id`（task_id）、`accepted`（bool）、`evidence`（text） | `review_task`（task_id）。独立レビューが必要なタスクを受け入れる場合は検証条件により必須 |
| `cancel` | `id`（task_id）、`reason`（text） | 準備済みで未割り当てのタスクに限る |
| `stop` | なし | なし |
| `snapshot` | `writers_quiescent`（bool）、`next_action`（text） | `files`（strings、[]）。書き込み停止フラグは true が必要 |
| `restore` | `destination`（text） | プロジェクトの外にある新しい保存先でなければならない |
| `halt` | `inventory`（inventory）、`observed_at`（timestamp）、`owned_processes_stopped`（bool）、`stop_evidence`（text） | `degraded`（bool、false）。true のとき `reason` と `next_action`（いずれも text）が必須。プロセス停止フラグには true が必要 |
| `resume` | `authorization`（authorization）、`interpretation`（string）、`source_message`（source）、`app_goal_observation`（app_goal） | 追加フィールドなし。停止確定後の対応する再開指示、新しい利用量と実行一覧、現在の検証済みチェックポイントが必要 |
| `compact` | なし | 受け入れ済みまたは拒否済みの終端タスク、稼働中の子エージェントがいないこと、現在の検証済みチェックポイントが必要 |
| `complete` | なし | compact の条件に加え、実行がアクティブであることが必要 |
| `status` | なし | 既存の有効化済み状態が必要 |

パスはすべてプロジェクトに対する厳密な相対ファイルパスです。Windows の別名、境界、シンボリックリンクに関する従来の保護を適用します。役割名と推論強度の文字列は既存の役割カタログと利用可否の確認に従います。読み取り専用の verifier や coordinator では書き込み対象を空にできます。存在しないタスクの参照はエラーになり、タスクを暗黙に作成することはありません。`accepted: false` は作業が停止しているタスクを拒否として終えます。`accepted: true` は必要なレビュー条件を通過した後で受け入れます。

<a name="raw-usage-lifecycle"></a>

## 生の使用量データの処理

N06 の `context_pin` は `{path, sha256, manifest_path, manifest_sha256}` の四つのフィールドから成ります。パスは空でないプロジェクト相対パス、SHA-256 は小文字です。マニフェストは既存の変更不可のタスク入力ファイルでなければなりません。`prepare` は、version1 の新しい要約表示を現在の状態と別途固定したマニフェストに照らして検証します。要約を使って準備したメッセージはこの手順にひも付きます。`dispatch-start` はさらに `current_context` を受け取り、prepare 後の状態と元のマニフェストで作り直した要約を要求します。ネイティブの制約はすべて有効です。要約生成、証跡による停止条件、生成側の検証、制約については [コンテキスト](context.md) を参照してください。

N07 の `status` は従来のタスク状態マップに `task_observations: {contract_version: 1, tasks: ...}` を追加します。ネイティブ実行の完了と成果物の受け入れは別であり、done でも finish の条件が通るまでは受け入れ待ちです。独立レビューを伴う完了では、従来の任意フィールド `review` にタスクへのリンクを記録し、生成・レビュー双方に異なる空でないネイティブセッション識別子を求めます。割り当て時のレビュー対象の出力ハッシュと、記録されたものも一致していなければなりません。schema2 は変わりません。レビューを、自分が固定した生成側のタスクの受け入れに直接・間接に依存させることはできません。過去に受け入れた独立レビュー済みの作業でも、生成・レビュー双方の識別情報やレビューの固定情報が不完全なら、`historical_unrecorded` として扱います。prepare、record、dispatch-start での新しい受け入れ依存は満たせません。要約側のブロック判定と補助プログラムは同じ独立性判定を使います。過去分の自動移行はありません。

呼び出し側の型や範囲が不正であればエラーとし、状態のバイト列を保持します。例として `usedPercent: "10"`、真偽値で渡した件数やタイムスタンプ、101 パーセント、負のリセット時刻、枠を表すリスト、有限でないメタデータが該当します。`rateLimits`、枠、計測値の欠落や null、型が正しくても 300 分または 10080 分のどちらかが欠けている計測、その他の点は有効でも古すぎるか未来の観測は、使用量を取得できなかった扱いです。このような形式上有効な `usage` コマンドは、意図的に古い使用量を消去し、`usage_errors` を増やします。初回は `RETRY_USAGE_ONCE`、2 回目は `SAVE_AND_STOP` を返し、すでに停止確定または完了済みの場合を除いて停止処理中にします。停止確定済みの実行を黙って再開することはありません。0、100、および範囲内の有限な小数は有効な計測値で、100 ではしきい値に応じた復旧処理が働きます。形式が壊れた使用量入力は、取得不能の回数に数えません。生のメタデータは保持しますが、必須の有効な計測枠の代わりにはなりません。

<a name="supported-pinned-entry"></a>

## 固定済みの正式なエントリーポイント

すべての操作に `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1` を使います。実行モードには `--project <absolute-project>` と `--run-id <fresh-run>` も必要です。

| モード | 入力 | 動作 |
| --- | --- | --- |
| `workflow validate` | 入力なし | 受け入れ済みパッケージと役割方針を検証し、非アクティブな名前空間にひも付ける |
| `workflow <typed-command>` | `--input <json-file>` または標準入力 | 上記の型付き仕様に従って実行する |
| `coordination` | `--input <json-file>` | モデルを使わない処理を範囲を限定してまとめて行う。ネイティブ割り当てを自動実行しない |
| `watcher` | `start`、`renew`、`stop`、`status`、`once` | 固定された内部の計測処理と、所有している実行状態を使う |
| `install` | 明示的な `--destination .../agent-workflow-v1-1`。`--update` は任意 | 同梱内容と、所有しているインストールの条件を検証する |

`workflow.py` の CLI を直接呼び出すと拒否されます。このファイルにはインストール、エクスポート、ネイティブの役割生成の入口はありません。配布ファイルのビルドには `tooling/build-candidate.py`、Codex へのインストールには固定済み候補パッケージのインストーラーだけを使います。旧スキーマの変換は候補パッケージの CLI コマンドにはなく、廃止されたインストールの起動手段にもしてはいけません。

<a name="input-and-state-boundary"></a>

## 入力と状態の境界

正しい真偽値を使うコマンドの判定は、使い捨ての対象で再現できます。任意フィールドでは安全な false の既定値を引き続き使えます。真と解釈できても型が不正な値は拒否します。状態を厳密に読み取ることで、JSON 構文と数値の有限性を守ります。文字列の権限情報と旧版の状態は拒否します。テストは使い捨てのプロジェクトだけを使います。

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## 型付きの割り当てと出所の時系列（A04／GI01）

N02 の `bind-active-goal` は、準備に時間がかかった後でも、ユーザーの発言を作り上げたり元の時刻を書き換えたりせず、現在アクティブなネイティブゴールをひも付けます。状態が存在しないか、厳密に `{schema_version: 2, status: inactive, tasks: {}}`（実際の JSON では文字列を通常どおり引用符で囲む）の場合に限ります。移行済み、完了済み、停止済み、停止確定済み、操作履歴ありなど、初期状態でない補助プログラムではひも付けできません。初回の観測は 0〜60 秒以内の実際の `get_goal` の証跡です。ゴール、スレッド、目的、元の作成時刻、観測時刻はひも付けと一致する必要があります。ネイティブゴールの作成はどれだけ古くても構いませんが、観測より後にはできません。コマンドの目的と objective の範囲は同じとし、報告用の範囲や拡大した範囲は拒否します。解釈と手順確認には空でない別々の証跡を渡します。これらはメイン側が現在の文脈と新しい制限を調べたことを記録するもので、補助プログラムは確認の真正性を保証できません。

状態は現在の根拠として `native_binding` を保存し、`native_binding_events` と `native_binding_ids` を人による権限の台帳とは分けて管理します。各イベントは `binding`、`interpretation`、`procedural_check`、元の `app_goal_observation` を含みます。ひも付け ID は後続の人のイベント ID と衝突しないよう予約します。実行と範囲の ID も既存履歴と合わせて予約します。この根拠で作業するには、目的と作成時刻の一致する新しいアプリゴール観測が必要です。人による権限については、出所の厳密なスキーマと元の発言から 300 秒以内という規則を保持します。人が再開すると現在のネイティブ根拠は置き換わりますが、ネイティブの履歴は別途保持し、その後に人が有効化する場合も保持します。人による再開後も含め、ネイティブの記録や予約に形式不正、欠落、重複、矛盾があれば拒否します。停止、スナップショット、停止確定の規則は引き続き適用されます。アクティブなアプリゴールを新たに観測しても、停止した補助プログラムは自動再開しません。

Schema 2 は廃止済みのタスクフィールド `inputs` を拒否します。再検証の際、過去の文字列を明示的に `context_refs` へ移せます。コンテキストの文字列をファイルやタスクの参照として解釈することはありません。`file_inputs` は厳密に `{path, revision}` のオブジェクトのリストです。割り当てではプロジェクト相対パスを正規化し、大文字・小文字を区別しない場合の重複、範囲外への脱出、予約済みパス、リンクを拒否し、`sha256` と `present` を固定します。欠落は null と false で記録します。変更不可の入力ファイルと同じタスクが所有する書き込み先を重ねることはできません。書き込み先には割り当て前の基準ハッシュを別に保持します。タスクには明示的なリビジョンと、その仕様全体を覆う割り当てハッシュがあります。入力やタスクのリビジョン、固定した値を入れ替えて黙ってハッシュを再計算しても、以前の成果を受け入れることはできません。

`review_targets` は、厳密に `{producer_id, revision}` のオブジェクトのリストで、verifier だけが使えます。生成側タスクはすべて指定リビジョンで存在し、割り当て時とレビュー時の両方で、そのセッションが idle、done、stopped のいずれかであるという新しい観測が必要です。割り当ては生成側の出力ファイルの正確なハッシュと有無を固定します。コンテキストへの参照はレビュー権限になりません。準備済みの引数は型付きの固定情報をすべて含みます。入力ファイル、生成側の仕様と出力は prepare、record、verifier の finish、生成側の受け入れ時に再検証します。受け入れ済みの依存も再確認します。A05 の accepted／running の状態遷移は変わりません。独立レビューを必要とする生成側タスクは、レビュー中に状態が running のままでも、実際には作業停止中と観測される場合があります。

割り当て仕様が古くなった場合は、`INPUT_INVALIDATED` とともに `invalidation` の記録を返します。この記録には task_id、割り当てのリビジョン、理由、該当する場合は変更された正確なパス、想定と実際のハッシュまたはリビジョン、at、ソートした affected_task_ids が入ります。同じパスを入力とするタスクが無効化の起点になるのは、そのタスク自身の固定ハッシュ、ファイルの有無、または割り当て仕様が観測値に合わない場合に限ります。現在のバイト列または欠落状態に正しく固定した新しいリビジョンは、古い固定値が無効になっても有効です。その後、実際の依存とレビューの関係をたどります。コンテキストの文字列やパスの一致だけで関係を作りません。運用状態はそのままにし、`contract_invalid: true` と永続化した記録で古い出力やレビューの受け入れを防ぎます。古いバイト列に戻しても、以前の生成側タスクや受け入れ済みのレビューは復活しません。判定を再度行っても、記録と状態のバイト列は保持します。

入力、レビュー対象の出力、依存先の出力を観測するときは、シンボリックリンク、ジャンクション、ハードリンクされたファイル、ディレクトリへの置換、範囲外への脱出をハッシュ計算前に拒否します。危険な出力が見つかると、レビューの割り当て、prepare、record、finish、生成側の受け入れの各時点で、エラーを返す前に生成側と影響を受けるレビュー・依存先を無効にします。記録には想定ハッシュ（または null）、正確な相対パス、管理された観測理由を `unsafe: ` で始めた `actual` 文字列として保持します。ファイルが存在しない場合は null を記録します。EIO など関係のない入出力障害はそのまま返し、仕様の変化だと誤って分類しません。確認するのは観測時のパスです。外部から同時に置き換えられる場合にファイルシステム全体のアトミックなスナップショットを提供するものではありません。

関係のない部分的な変更はすべて破棄します。直接 `apply` を実行すると、渡された状態に無効化の記録だけを残します。execute は例外を返す前にそれをアトミックに保存します。リセットや再ハッシュのコマンドはありません。記録を確認し、新しいタスク ID とリビジョンに新しい固定情報を付け、明示的に割り当ててください。他の条件で拒否された場合は従来どおり状態のバイト列を保持します。

GI01 は、権限情報と、それに厳密に対応する source_message の両方に source_created_at と original_turn_id を求めます。observed_at と observed_turn_id は観測を別に記録します。出所の作成が観測に先立ち、新規受け入れ時に作成から 300 秒以内であることが条件です。再開では出所の作成時刻が停止、停止確定、完了より後でなければなりません。観測時刻が後であるだけでは足りません。最後に受け入れた出所の基準時刻は再開や新規の有効化後も保持されます。作成時刻をさかのぼれず、同時刻なら元のターンの一致が必要です。イベント ID と元の出所 ID は、種別、観測時刻、実行、スレッド、再開をまたいで再利用しないよう予約します。観測イベントは現在のゴール、実行、スレッドと一致させます。明示的な停止には目的の範囲の一致も必要です。将来の意図、方針、報告の観測は証跡として保存できますが、現在の目的を実行したり再開したりする権限にはなりません。使用量のしきい値による停止は補助プログラムによる機械的な処理です。アプリゴールの観測は変えず、ユーザーから一時停止の権限を得たことにもなりません。

これらのフィールドは渡された証跡であり、プロバイダーの認証ではありません。この補助プログラムはネイティブのゴールツールの直接呼び出しを横取りせず、自分で update_goal を呼び出すこともありません。過去の停止、新しい再開、同じターンでのコンテキストの再送を順に行う制御された回帰テストでは、補助プログラムが再送を拒否し、アプリ側のアクティブな観測は変わらないことを確認しています。N07 は上記の、版を持つ実行と受け入れの観測ビューを追加します。新規のネイティブ子エージェントでのコンテキストおよびキャッシュの計測と、A06／GI02 の一時停止の統合検証には、引き続き別の証跡が必要です。
