<a name="state-compatibility-and-recovery-boundaries"></a>

# Zustandskompatibilität und Grenzen der Wiederaufnahme

Die Laufzeit v1.1 akzeptiert nur aktuellen schema2-Zustand in einem freigegebenen, unveränderlichen Lauf-Namensraum. Unbekannter, beschädigter und älterer Zustand wird abgelehnt, ohne den operativen Zustand zu verändern. Alte Freigabetexte, archivierte Aufgabenbelege und veraltete Zielbeobachtungen erteilen keine Freigabe.

Der abgesicherte Einstiegspunkt `scripts/candidate.py` bietet keinen Migrationsbefehl. Der Kern enthält weder einen Konverter für alten Zustand noch einen Prüfer für alte Freigaben. Automatischer Import, Start auf Basis einer anderen Installation und Rückkehr zu deren Paket werden nicht unterstützt. Beginne nach einem freigegebenen Paketupdate einen neuen Lauf. Das Manifest und die Zuordnung zum Hilfsprogramm müssen im laufenden Lauf gleich bleiben.

Die aktuelle Wiederaufnahme verwendet den verifizierten Kontrollpunkt des angenommenen Laufs und eine neue, gültige Freigabe des Nutzers. Erhalte die Quellenzeitstempel, die gegen Wiederverwendung reservierten Identitäten, die Dateiintegrität und den Nachweis, dass die schreibenden Agenten angehalten haben. Alte Stopp- oder Fortsetzungstexte bleiben historische Belege. Das Hilfsprogramm aktiviert das wiederhergestellte Ziel nicht automatisch.

Neue Kontrollpunkte gehören `agent-workflow-v1-1`. Das eigenständige Standard-Zustandsverzeichnis des Kerns ist `.agent-workflow-v1-1`. Der unterstützte, abgesicherte Einstiegspunkt verwendet `.agent-workflow-v1-1/runs/<run-id>`. Die früheren Verzeichnisse `.agent-workflow` und `.agent-workflow-v1-0` werden nicht importiert. Sie bleiben reserviert und dürfen nicht als Aufgabenpfad oder Wiederherstellungsziel dienen. Die aktuelle Snapshot-Bereinigung lässt ihre Besitzmarkierungen bestehen. Ein direkter Aufruf der Kern-CLI wird verweigert.

<a name="move-the-user-installation-to-the-11-technical-identity"></a>

## Benutzerinstallation auf die technische Identität 1.1 umstellen

1. Ermittle laufende Prozesse und Agenten mit Schreibzugriff. Stoppe nur Arbeit, die zu deiner Ausführung gehört. Prüfe vor der Verschiebung einer älteren Installation den zugehörigen Stopp und das tatsächliche Prozessende.
2. Gleiche das alte Paket mit seinem Besitznachweis ab. Prüfe die Hashes einer Sicherung außerhalb aller Verzeichnisse, in denen Skills gefunden werden. Erhalte bestehende Änderungen, bevor du weiterarbeitest.
3. Installiere das neue Paket frisch unter `skills/agent-workflow-v1-1`, mit dem neuen Manifest und der separat gesicherten Prüfsumme. Verwende `--update` nicht für eine frühere technische Identität.
4. Führe vor der Änderung der Suchpfade `workflow validate` vom neu installierten Einstiegspunkt aus. Verwende dafür eine neue, eindeutige Prüf-Lauf-ID. Vergleiche die zurückgemeldete Identität, Version, den Arbeitsbereich und den Manifest-Hash. Das erzeugt `run-binding.json`, ohne Ziel, Workflow-Zustand oder Monitor zu starten.
5. Der neue Aufruf lautet `$agent-workflow-v1-1`. Aktualisiere nach der direkten Validierung die Pfade des Haupt-Skills und der Rollen-Skills in Codex. Behalte die Aktivierungs- und Aufrufregeln bei und schütze parallele Konfigurationsänderungen. Prüfe die zugehörigen Prozesse erneut und erhalte aktive Installationen, die andere Sitzungen nutzen. Archiviere das alte Paket nach der sicheren Umstellung außerhalb der Skill-Suchpfade. Prüfe anschließend Konfiguration und Sicherung.
6. Erhalte Identitäten, Pfade und Hashes in alten Kontrollpunkten, Laufbindungen und Belegen. Beginne mit dem neuen Paket einen neuen Lauf. Übergib bei relevanter unfertiger Arbeit die geprüften früheren Ergebnisse und die tatsächlich aktuelle Freigabe getrennt.
