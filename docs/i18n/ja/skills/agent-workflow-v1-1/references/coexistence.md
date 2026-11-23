<a name="pinned-runtime-binding-and-installation"></a>

# 固定した実行の紐付けとインストール

技術識別子は `agent-workflow-v1-1`、表示名は `agent-workflow-v1.1`。暗黙の呼び出しは無効で、明示的な選択だけがこの candidate を採用します。メインのモデルはユーザーが選びます。ロールに使えるのは `gpt-6.1-sol`、`gpt-6-astra`、`gpt-6.1-terra` のいずれかで、推論は `medium` 以上です。非公開の監視助言担当は Terra/medium に固定し、自動で代替しません。

現在の明示選択後、メインエージェントが同じターンで対象の実行を起動して検証します。以前に無効化した実行を再度有効にできますが、過去の選択やパッケージ保守は起動許可になりません。完全な起動の証拠がない場合や、ネイティブ目標を作れない場合は[運用](operations.md)の手動監視を使ってください。起動確認前に実作業を始めず、その後の停止指示を優先します。

Python 3.11+ と標準ライブラリでビルドします。

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

出力は `agent-workflow-v1-1/`、決定的な ZIP、`build-receipt.json` です。受け入れた `manifest_sha256` は変更可能なパッケージの外に、調整や引き継ぎの証拠として保持します。呼び出し元が提示する固定値が信頼の起点です。実行への紐付け、重要な状態書き込み、調整、測定の前にマニフェストと依存関係を検証します。ハートビートでは実行・エポック制御だけを確認し、毎秒全ファイルを巡回しません。編集後にハッシュを再計算しても外部の承認にはなりません。Python が検証するのはバイトと識別情報で、ユーザーの承認やプロバイダー、モデルのメタデータ、呼び出し側が管理するファイルシステムは認証できません。実行ファイルやインタープリターが侵害されていたり、承認済み外部ハッシュを紐付けと一緒に改変できたりする場合は範囲外です。

グローバル引数はモード名の前です。ソースから実行するときは `skills/agent-workflow-v1-1/scripts/candidate.py` と生成したマニフェストを使います。インストール後は導入先の `scripts/candidate.py` とマニフェストに、保存した同じ外部ハッシュを渡します。それぞれ別の実行 ID を使用します。不変の紐付けは依存関係とマニフェストの解決後パス、プロジェクト、実行、版、ハッシュを保持します。入口の差し替え、パッケージの交換、依存関係の編集・欠落、別のハッシュでの実行再利用は拒否されます。承認済み更新後は新しく実行します。

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

許可された `workflow activate` または `bind-active-goal` は監視を起動し、`--usage-input` による最初の正常なサンプルを確認します。`resume` は条件を満たす続行、または許可済みの有効な目標で期限切れから復旧する場合に、新しい受領記録で別エポックを開きます。[運用](operations.md)と[ネイティブ使用量](native-usage.md)を参照してください。測定で別サーバーや CLI は起動しません。有効なコマンドはリースを更新し、`complete`/`halt` は所有 Python プロセスを停止して終了を確認します。明示的な `watcher` コマンドも同じ受領記録を使います。

`workflow` はコアの型付きコマンドと `validate` を受け付けます。JSON は `--input` か stdin から読みます。旧コア CLI の `native/install/export/migrate` は非対応です。`activate` の許可と `bind-active-goal` の証拠には選択した実行 ID が必要です。`validate` はロールと依存関係を確認し、領域を紐付けますが目標は有効にしません。バッチ例は `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`。最大 64 手順、メインだけが書き込み、未決結果を保存します。自動再試行やネイティブ割り当ては行いません。実作業には型付き許可と受け入れが必要です。終了一覧だけでは成果物は合格しません。書き込みの止まった古いタスクは `finish accepted:false` で破棄できます。ライブの書き込み担当は破棄できません。圧縮は検証済みチェックポイントを要求し、タスクと無効化履歴を保存します。

隔離ルートは `<project>/.agent-workflow-v1-1/runs/<run-id>/`。実行状態の領域はタスクの書き込み権から除外します。統合監視はそこにある `watcher/`、固定スケジューラー、変更不可の所有情報、candidate 入口の再起動引数を使用します。他実行の制御や私用 CLI を使って別プロセスのリース更新や停止はできません。応答のないチャットはリースを延長しません。作業の区切りで有効なコマンドを使います。更新は実行中プロセスを移しません。別の監視スキルやランチャーはなく、承認済みパッケージから依存関係を読みます。

新規、変更不要、管理下の更新には、末尾が `agent-workflow-v1-1` の明示的なインストール先が必要です。

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

更新できるのは所有記録とバイトが一致するインストールです。置換前にステージングを検証します。失敗すると元ディレクトリを復元し、成功するとトランザクション資料を片付けます。恒久的なロールバック用アーカイブも専用コマンドもありません。復元自体が失敗した場合はトランザクションディレクトリに唯一の旧コピーが残るため、復元確認まで保管します。これらのコマンドはファイルだけを準備して `activated:false` を返します。カタログ表示、明示的なネイティブスキル選択、プロバイダー上の実効モデル、実際の成果物の受け入れは別に確認します。

Windows では仮ステージングの ACL をインストール先の親から継承するようにしてからコピーします。アカウント名は固定しません。ユーザー別の `skills` 親ディレクトリが読み取りと実行を許可する必要があります。導入後、昇格していない通常の Codex セッションで検出と candidate の直接実行を確認してください。管理者セッションだけでの成功では十分ではありません。
