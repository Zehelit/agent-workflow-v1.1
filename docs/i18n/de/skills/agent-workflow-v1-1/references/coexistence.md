<a name="pinned-runtime-binding-and-installation"></a>

# Abgesicherte Laufzeitbindung und Installation

Die technische Identität lautet `agent-workflow-v1-1`, der Anzeigename `agent-workflow-v1.1`. Der implizite Aufruf ist deaktiviert. Dieser Kandidat wird ausschließlich durch ausdrückliche Auswahl verwendet. Der Nutzer wählt das Modell des Hauptclients. Workflow-Rollen dürfen ausschließlich `gpt-6.1-sol`, `gpt-6-astra` oder `gpt-6.1-terra` mit mindestens `medium` verwenden. Die interne Überwachungsrolle ist auf Terra/medium festgelegt. Ein Ausweichmodell ist nicht zulässig.

Eine aktuelle ausdrückliche Auswahl verpflichtet den Hauptagenten, den abgegrenzten Lauf im selben Turn zu starten und zu überprüfen. Sie kann diesen Lauf nach einer früheren Deaktivierung erneut aktivieren. Eine ältere Auswahl und Paketwartung können das nicht. Fehlen die Belege für eine vollständige Aktivierung oder darf kein natives Ziel angelegt werden, verwende den manuellen Überwachungsstart aus dem [Betriebsleitfaden](operations.md). Beginne erst nach überprüftem Start mit der eigentlichen Arbeit. Weiche nicht still auf direkte, unüberwachte Arbeit aus. Spätere Stoppanweisungen haben weiterhin Vorrang.

Baue mit Python 3.11 oder neuer und der Standardbibliothek.

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

Die Ausgabe enthält `agent-workflow-v1-1/`, ein reproduzierbares ZIP und `build-receipt.json`. Bewahre `manifest_sha256` aus dem Beleg zusammen mit den angenommenen Koordinations- oder Übergabebelegen außerhalb des veränderlichen Pakets auf. Der Aufrufer übergibt diesen Hash als Vertrauensanker. Manifest und Abhängigkeiten werden vor der Laufbindung, vor wichtigen Workflow-Zustandsänderungen, vor Koordinationsschritten und bei jeder Messung des Prozesses vollständig geprüft. Die Veröffentlichung eines Lebenszeichens kontrolliert nur die kleinen Steuerdaten von Lauf und Epoche. Sie durchläuft und hasht das Paket nicht jede Sekunde. Wer den SHA eines veränderten Manifests neu berechnet und übergibt, hat keine externe Bestätigung erbracht. Python prüft Byteintegrität und Identität. Es kann weder die menschliche Freigabe noch Anbieter- und Modellmetadaten oder ein vom Aufrufer kontrolliertes Dateisystem authentifizieren. Kompromittierte ausführbare Dateien oder Interpreter und Aufrufer, die Bindung und freigegebenen externen Hash zugleich ändern, liegen außerhalb dieser Integritätszusagen.

Globale Optionen stehen vor dem Modus. Ein Lauf aus dem Quellstand nutzt `skills/agent-workflow-v1-1/scripts/candidate.py` und das gebaute Manifest. Eine installierte Ausführung nutzt den installierten `scripts/candidate.py` und sein Manifest mit demselben separat gesicherten Hash. Für jede Zuordnung aus Quelle oder Installation gibt es eine eigene Lauf-ID. Die unveränderliche Bindung enthält aufgelöste Abhängigkeits- und Manifestpfade, Projekt, Lauf, Version und Hashes. Die Ausführung wird verweigert, wenn sie zu einem anderen Einstiegspunkt verschoben wird oder ihr Paket ersetzt wurde. Sie wird auch bei bearbeiteten oder fehlenden Abhängigkeiten oder bei Wiederverwendung der Lauf-ID mit abweichendem Hash verweigert. Nach einem freigegebenen Update beginnt ein neuer Lauf.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

Ein freigegebenes `workflow activate` oder `bind-active-goal` startet den Monitor und verifiziert seinen ersten gültigen Messwert, der über `--usage-input` geliefert wird. `resume` öffnet nach einer zulässigen Fortsetzung oder nach kontrollierter Wiederaufnahme wegen abgelaufener Messung eine neue Epoche mit einem neuen nativen Beleg. Bei der technischen Wiederaufnahme bleibt die Freigabe des aktiven Ziels maßgeblich. Siehe [Betrieb und Wiederaufnahme](operations.md) und [native Nutzungsbelege](native-usage.md). Für Messungen wird kein Codex-Server und keine Codex-CLI gestartet. Aktive Befehle verlängern die Laufberechtigung. `complete` und `halt` stoppen den zugehörigen Python-Prozess und verifizieren sein Ende. Für ausdrückliche `watcher`-Befehle gelten dieselben Beleganforderungen.

`workflow` akzeptiert die typisierten Zustandsbefehle des Kerns und `validate`. Befehle lesen JSON mit `--input` oder über die Standardeingabe. Die alten Kern-CLIs `native/install/export/migrate` werden für diesen Kandidaten nicht unterstützt. Sowohl die Freigabe für `activate` als auch die Belege für `bind-active-goal` müssen die gewählte Lauf-ID enthalten. `validate` kontrolliert Rollen und Abhängigkeiten und bindet den Namensraum, ohne ein Ziel zu aktivieren. Ein Stapel sieht so aus: `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`. Er führt höchstens 64 Schritte aus. Der Hauptagent ist der einzige Schreiber. Ausstehende Ergebnisse werden dauerhaft festgehalten und weder automatisch wiederholt noch nativ versendet. Inhaltliche Befehle erfordern weiterhin aktuelle typisierte Freigaben und passende Abnahmebelege. Ein Sitzungsbestand mit Endstatus nimmt keine Ausgaben ab. Veraltete Aufgaben ohne aktive Schreibprozesse können mit `finish accepted:false` verworfen werden. Solange jemand schreibt, ist das nicht möglich. Die Komprimierung des Zustands verlangt einen verifizierten Kontrollpunkt und archiviert die Historie der Aufgaben und Ungültigkeiten.

Das isolierte Stammverzeichnis lautet `<project>/.agent-workflow-v1-1/runs/<run-id>/`. Die Stammverzeichnisse operativer Zustände sind reserviert und können keinen Aufgaben als Schreibpfade zugewiesen werden. Der integrierte Monitor verwendet dort `watcher/`, einen ausdrücklich fixierten Ablaufplaner, unveränderliche Zugehörigkeitsdaten und die Argumente für einen Neustart über den Kandidaten-Einstiegspunkt. Steuerdaten eines falschen Laufs oder der direkte Zugriff auf die interne CLI können die Berechtigung eines anderen Prozesses weder verlängern noch ihn anhalten. Ein stiller Chat kann die befristete Berechtigung nicht verlängern. Verwende an Arbeitsschritten einen aktiven Kandidatenbefehl. Paketupdates migrieren laufende Prozesse nicht stillschweigend. Es gibt keinen eigenen Monitor-Skill und keinen eigenen Launcher. Private Abhängigkeiten laden aus dem freigegebenen Paket.

Erstinstallation, Prüfung ohne Schreibvorgang und reguläres Update brauchen ein ausdrückliches Ziel, das auf `agent-workflow-v1-1` endet.

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

Updates setzen voraus, dass die verwaltete Installation unverändert ist. Der Installer verifiziert ein vorbereitendes Paket, bevor er das bestehende Verzeichnis ersetzt. Bei einem Fehler beim Dateiaustausch wird das vorherige Verzeichnis wiederhergestellt. Bei erfolgreichem Update werden die Transaktionsdateien entfernt. Es gibt weder ein dauerhaftes Rücksicherungsarchiv noch einen Rücksetzungsbefehl. Schlägt auch die Wiederherstellung fehl, enthält das Transaktionsverzeichnis die einzige vorherige Kopie für eine ausdrückliche Wiederherstellung. Lösche es erst, wenn die Wiederherstellung geprüft ist. Die Befehle bereiten nur Dateien vor, erkennbar an `activated:false`. Ob der Skill im Katalog erscheint, sich nativ ausdrücklich auswählen lässt, welche Modellmetadaten der Anbieter tatsächlich liefert und ob das Produkt in der Praxis funktioniert, wird nach der Installation separat geprüft.

Unter Windows setzt der Installer vor dem Kopieren die Zugriffssteuerung des temporären Verzeichnisses so zurück, dass sie vom Ziel-Elternverzeichnis erbt. Es ist kein bestimmtes Windows-Konto vorgegeben. Ein anderer oder umbenannter Benutzer erhält die Rechte, die seinem Elternverzeichnis `skills` bereits zugewiesen wurden. Das Elternverzeichnis muss dem vorgesehenen Benutzer Lesen und Ausführen erlauben. Prüfe nach der Installation, ob Codex den Skill findet und ob der direkte Aufruf des Kandidaten in einer normalen Sitzung ohne erhöhte Rechte funktioniert. Der Erfolg einer erhöhten Sitzung allein reicht nicht aus.
