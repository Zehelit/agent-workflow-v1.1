<a name="changelog"></a>

# Änderungsprotokoll

Dieses Protokoll enthält die vorhandene Versionsgeschichte. Der Quellstand und der freigegebene Build-Beleg identifizieren das genaue Paket.

<a name="11--2026-11-10"></a>

## 1.1 — 10. November 2026

<a name="added"></a>

### Neu

- Dauerhafter Nachweis für den ersten Aufruf von activate oder bind-active-goal. Eine sichere Wiederholung stützt sich auf den außerhalb des veränderlichen Pakets aufbewahrten SHA-256 des Fehlerbelegs.
- Ein dem Lauf zugehöriger Schreiblock für die Lebenszyklusoperationen des Kandidaten. Nur ein Prozess darf ihn zum Schreiben halten.
- tooling/run-lifecycle-check.py für Vorbereitung, Start, Aktualisierung der Beobachtungen, Wiederaufnahme nach einem Stopp und regulären Abschluss.
- Ausdrücklich aufzurufender Modus timed-test mit getrennten Zeitangaben für SAVE, Kontrollpunkt und Stopp sowie einer Einstufung verspäteter Ereignisse. Das Werkzeug weist keine native Pause nach.
- Tatsächlicher Messzeitpunkt, Frist, Intervall und ein Vorschlag zur früheren Aktualisierung in den Statuszusammenfassungen.
- Dokumentation zu Zweck, Architektur, Workflow und Orchestrierung mit Diagrammen, Installationsanleitungen für Menschen und Agenten sowie ein Leitfaden zum Lebenszyklus.

<a name="fixed"></a>

### Korrigiert

- Kontrollierte Wiederaufnahme bei abgelaufenem Nutzungsbeleg mit demselben autorisierten und weiter aktiven Ziel. Tatsächlich ausgeschöpftes Kontingent und ausdrückliche Stopps durch den Nutzer haben eigene Wege.
- Die Wiederholung nach einem fehlgeschlagenen Erststart erhält Umfang, frühere SAVE-Belege, Paketbindung und die Prüfung, dass der Prozess nicht mehr läuft.
- Eine zeitlich begrenzte Ausnahme für das Wochenfenster gilt im Workflow-Kern und im Monitor gleich. Beobachtete Werte bleiben erhalten.
- Begrenzte Wiederholungen behandeln erkannte vorübergehende Windows-Freigabe- und Lesefehler bei Sperrdateien. Permanente oder nicht eingestufte Fehler bleiben sichtbar.
- Präzisere Regeln für die Aktualisierung von Messwerten, die Verlängerung der Laufberechtigung und die Aktualität der Belege nach einem Stopp.
- Überarbeitete Aufrufsignaturen älterer Testhilfen und Wartevorgänge für Prozesse und Dateien.

<a name="compatibility-and-evidence"></a>

### Kompatibilität und Nachweise

- Aufrufname, Installationsordner und Laufzeit-Namensraum wechseln gemeinsam auf 1.1. Sie heißen `$agent-workflow-v1-1`, `agent-workflow-v1-1/` und `.agent-workflow-v1-1/runs/`.
- Neue Manifeste lehnen die frühere technische Identität ab. Alte Läufe bleiben als historische Nachweise erhalten und werden nicht automatisch neu gebunden. Ihre Namensräume bleiben für Zuständigkeitsnachweis und Wiederherstellung reserviert.
- Ein Update überschreibt keine unveränderliche Paketbindung eines Laufs. Beginne nach der Freigabe eines neuen Pakets einen neuen Lauf.
- Python 3.11 oder neuer und ausschließlich die Standardbibliothek. Der Monitor erreicht native Nutzungswerkzeuge nicht selbstständig und startet keinen zweiten Codex-Server.
- Isolierte Tests verwenden künstliche Host-Beobachtungen und echte Python-Prozesse. Remote, Continue und Zeitmessungen in der laufenden Oberfläche erfordern separate Beobachtungen.
- Diese Version wurde in Quellcode-Commits dokumentiert. Ein GitHub-Release und ein Tag lagen außerhalb des damals festgehaltenen Veröffentlichungsumfangs.

<a name="10--previous-source-version"></a>

## 1.0 — frühere Quellcode-Version

- Per Manifest-Hash abgesicherter Kandidaten-Einstiegspunkt, typisierter Workflow-Zustand, Rollendokumente sowie Kontrollen von Schreibzuständigkeit und Kapazität.
- Integrierter Monitor ohne Modell, Kontrollpunkte und Wiederherstellung, dauerhaftes Koordinationsjournal und reproduzierbarer Paketbau.
- Das historische Protokoll macht keine Aussage darüber, ob ein passendes GitHub-Release oder Tag existiert.
