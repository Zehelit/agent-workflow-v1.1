<a name="install-verify-and-maintain-agwf"></a>

# AGWF installieren, prüfen und pflegen

Verwende Python 3.11 oder neuer und Git. Das Bauwerkzeug und der Kandidat nutzen die Python-Standardbibliothek. Installiere in ein Skill-Verzeichnis, in das der vorgesehene Benutzer schreiben kann. Die folgenden Befehle verwenden PowerShell. Wähle unter Linux oder macOS das entsprechende Verzeichnis unter dem vorgesehenen Codex-Stammverzeichnis und passe die Shell-Syntax an. Die Prüfungen der Windows-Zugriffssteuerungslisten (ACLs) sind plattformspezifisch.

<a name="build-from-the-selected-checkout"></a>

## Paket aus dem gewählten Quellstand bauen

Führe die Befehle im Stammverzeichnis des AGWF-Repositorys aus. Halte den Quell-Commit fest und prüfe lokale Änderungen, bevor du den Build freigibst. Das Ausgabeverzeichnis muss neu sein.

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

Der Build erzeugt ein Paketverzeichnis, ein reproduzierbares ZIP und einen Build-Beleg. Bewahre den freigegebenen Beleg außerhalb des veränderlichen Pakets zusammen mit dem Nachweis zum Quell-Commit auf. Der Installer erwartet den freigegebenen Wert `manifest_sha256`. Ein nach einer unerwarteten Änderung berechneter Hash ist eine andere Beobachtung. [Paketbindung und Installation](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## Selbst installieren oder einen Agenten beauftragen

Für das Codex-Skill-Stammverzeichnis des aktuellen Benutzers verwende folgende Befehle. Setze das Ziel für eine Testinstallation auf ein isoliertes Verzeichnis, das auf `agent-workflow-v1-1` endet. Derselbe Befehl dient der Erstinstallation und der Überprüfung eines identischen Pakets ohne Änderungen.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` zeigt an, dass der Vorgang Paketdateien installiert oder überprüft hat, ohne einen Workflow zu starten. Unter Windows erbt das temporäre Verzeichnis die Rechte des Ziel-Elternverzeichnisses. Prüfe mit dem vorgesehenen normalen Benutzer, dass er die installierten Dateien lesen und ausführen kann.

Bei einer Installation durch einen Agenten gelten folgende Schritte.

1. Bestätige das gewünschte Ziel und kläre, ob es sich um eine Erstinstallation, ein Update oder einen isolierten Test handelt. Erfasse das gegenwärtige Paket, Benutzeränderungen und zugehörige Prozesse, bevor du ein bestehendes Ziel änderst.
2. Prüfe den Quellstand und die Python-Version. Baue in ein neues Verzeichnis und sichere den freigegebenen Manifest-Hash wie oben.
3. Verwende für das gewünschte Ziel den per Manifest-Hash abgesicherten Kandidaten-Installer. Stoppe vor einem Update nur die betroffenen Prozesse, die zur alten Installation gehören, und verifiziere ihr Ende. Verweigert der Installer den Vorgang, bleibt das vorhandene Ziel zur Prüfung erhalten.
4. Verifiziere das installierte Manifest und die Dateien. Führe dann die passive Prüfung unten vom installierten Einstiegspunkt aus. Melde den Quell-Commit, den Manifest-Hash, das Ziel und den resultierenden Status. Gib an, ob du passiv geprüft oder ausdrücklich einen echten Workflow gestartet hast.

<a name="verify-the-installed-entrypoint"></a>

## Installierten Einstiegspunkt prüfen

Verwende ein neues Wegwerfprojekt und eine neue Lauf-ID. Diese Prüfung schreibt eine Laufbindung in das Wegwerfprojekt, ohne einen Prozess zu starten oder ein Ziel zu aktivieren.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Prüfe die Antwortfelder `identity`, `version`, `run_id`, `manifest_sha256` und `missing_dependencies`. In einem isolierten Test für diese Dokumentation meldete der Installer `activated:false`. Die Validierung erzeugte `run-binding.json` ohne Workflow-Zustandsdatei oder Monitorverzeichnis. Ein falscher Manifest-Hash wurde mit Exitcode 2 abgelehnt.

Der nächste echte Codex-Aufruf erfolgt getrennt. Möglicherweise muss die offene Sitzung ihren Skill-Katalog aktualisieren. Wähle `$agent-workflow-v1-1` ausdrücklich aus und gib den gewünschten Arbeitsumfang an. Der [Betriebsleitfaden](../skills/agent-workflow-v1-1/references/operations.md) beschreibt den freigegebenen Start und die Überprüfung des Monitors.

<a name="update-recover-or-remove"></a>

## Aktualisieren, wiederherstellen oder entfernen

- Stoppe vor dem Update einer verwalteten Installation den betroffenen Prozess über seinen bisherigen abgesicherten Einstiegspunkt und prüfe sein Ende. Baue das neue Paket und gib es separat frei. Ergänze `--update` nur bei einer unveränderten, vom Installer verwalteten Installation. Kläre zuerst vorhandene Benutzeränderungen.
- Der Installer prüft vor dem Ersetzen das vorbereitete Paket und versucht bei einem Fehler, das bisherige Verzeichnis wiederherzustellen. Schlägt auch die Wiederherstellung fehl, behalte das Transaktionsverzeichnis. Ein erfolgreiches Update bewahrt kein dauerhaftes Rücksicherungsarchiv. [Details zum Installer](../skills/agent-workflow-v1-1/references/coexistence.md).
- Eine andere technische Identität erfordert die Neuinstallation im neuen Verzeichnis. Bewahre alte Laufbindungen mit ihren ursprünglichen Manifesten auf und kläre die Skill-Suchpfade getrennt. Beginne nach einem freigegebenen Paketupdate einen neuen Lauf. [Grenzen der Migration](../skills/agent-workflow-v1-1/references/migration.md).
- Es gibt keinen Deinstallationsbefehl. Prüfe vor der Entfernung das genaue Skill-Verzeichnis, beende die zugehörigen Prozesse und lege eine verifizierte Kopie außerhalb der Skill-Suchpfade ab. Erhalte Benutzerprojekte, ihren Laufzustand, Kontrollpunkte und externe Belege. Prüfe nachher, welche Skills gefunden werden.
