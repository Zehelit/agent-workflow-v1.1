<a name="install-verify-and-maintain-agwf"></a>

# AGWF のインストール、検証、保守

Python 3.11 以降と Git が必要です。ビルドと candidate は Python の標準ライブラリだけを使います。対象ユーザーが書き込めるスキルディレクトリへインストールしてください。以下は PowerShell の例です。Linux や macOS では、対象の Codex ルートにある対応ディレクトリを使用し、シェルの構文を調整してください。Windows の ACL 確認はそのプラットフォーム固有です。

<a name="build-from-the-selected-checkout"></a>

## 選択した作業コピーからビルドする

AGWF リポジトリのルートから実行します。ソースコミットを記録し、ビルドを受け入れる前にローカル変更を確認します。出力先には新規ディレクトリを指定します。

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

ビルダーはパッケージディレクトリ、同じ入力から再現できる ZIP、ビルド記録を生成します。受け入れた記録は書き換え可能なパッケージの外で、元のコミットの証拠と一緒に保管します。インストーラーには受け入れた `manifest_sha256` を渡します。想定外の変更後にハッシュを再計算した値は別の観測です。[パッケージの紐付けとインストール](../skills/agent-workflow-v1-1/references/coexistence.md)。

<a name="choose-a-human-or-agent-path"></a>

## 手動かエージェント経由でインストールする

現在のユーザーの Codex スキルルートに対して次を実行します。一時的な試験なら、末尾が `agent-workflow-v1-1` の隔離先に変更します。新規インストールも、同一パッケージを変更せずに確認する場合もコマンドは同じです。

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` はファイルだけをインストールまたは検証したことを示します。ワークフローは開始しません。Windows ではステージングディレクトリがインストール先の親から権限を継承します。実際に使う通常ユーザーで読み取りと実行を確認してください。

エージェントが実行する場合：

1. 指定先と、新規・更新・隔離試験のいずれかを確認します。既存ディレクトリを変える前に現在のパッケージ、ユーザー編集、所有プロセスを把握します。
2. 作業コピーの識別情報と Python のバージョンを検証します。新しいディレクトリにビルドし、承認されたマニフェスト固定値を外部に保管します。
3. 指定先へ固定した candidate インストーラーを使います。更新前は対象に属する旧プロセスだけを停止して終了を確認します。拒否されたときは元のディレクトリを調査用に残します。
4. インストール済みのマニフェストとファイルを検証し、インストールされた入口から下の受動検証を行います。元コミット、マニフェストハッシュ、インストール先、状態、受動検証か明示的に起動した実際のワークフローかを報告します。

<a name="verify-the-installed-entrypoint"></a>

## インストール済みの入口を検証する

新しい一時プロジェクトと実行 ID を使います。これは一時プロジェクトに実行の紐付けを書きますが、監視プロセスや目標は起動しません。

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

応答の `identity`、`version`、`run_id`、`manifest_sha256`、`missing_dependencies` を確認します。この文書用の隔離試験ではインストールが `activated:false` を返し、検証が `run-binding.json` を作成しました。ワークフロー状態ファイルも監視ディレクトリも作成されず、誤ったマニフェストハッシュは終了コード 2 で拒否されました。

実際の Codex 呼び出しは別の手順です。開いているセッションのスキルカタログに再読み込みが必要な場合があります。`$agent-workflow-v1-1` を明示的に選択し、作業範囲を伝えてください。起動許可と監視状態の確認は[運用ガイド](../skills/agent-workflow-v1-1/references/operations.md)に従います。

<a name="update-recover-or-remove"></a>

## 更新、復旧、削除

- 管理下のインストールを更新する前に、既存の固定入口から対象プロセスを停止し、実際に終了したことを確認します。新しいパッケージを別にビルドし、受け入れます。所有と未変更を確認したインストールにだけ `--update` を追加します。ユーザーの編集があれば先に確認します。
- インストーラーは置き換え前にステージング済みのパッケージを検証し、置き換えに失敗した場合は前のディレクトリの復元を試みます。復元も失敗したらトランザクションディレクトリを残します。更新成功後に恒久的なロールバック用アーカイブは残りません。[インストーラーの詳細](../skills/agent-workflow-v1-1/references/coexistence.md)。
- 技術識別子が変わったら、新しいディレクトリに新規インストールします。以前の実行は元のマニフェストと一緒に保管し、スキルの検出設定は別に確認します。受け入れた新しいパッケージでは新しい実行を使用します。[移行の範囲](../skills/agent-workflow-v1-1/references/migration.md)。
- アンインストールコマンドはありません。削除前に対象の正確なディレクトリを確認し、その所有プロセスを停止して、スキル検出の対象外に検証済みのコピーを保存します。ユーザープロジェクト、実行状態、チェックポイント、外部の記録は保持します。変更後にスキルの検出を確認します。
