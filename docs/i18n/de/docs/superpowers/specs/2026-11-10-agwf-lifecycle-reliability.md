<a name="lifecycle-reliability-specification"></a>

# Spezifikation zur Zuverlässigkeit des Lebenszyklus

> Historische Spezifikation vom 10. November 2026. Die nachstehenden Belege und Freigaben gelten für die damaligen Arbeiten. Das aktuelle Verhalten beschreibt der [Betriebsleitfaden](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="intended-result"></a>

## Angestrebtes Ergebnis

Die erste Aktivierung, die Aktualisierung von Nutzungsdaten, die Wiederaufnahme und den regulären Abschluss zuverlässig machen. Den Test vor den geplanten Läufen vorbereiten. Die Ergebnisse von Start, Wiederaufnahme nach einem angehaltenen Workflow und Zeitprüfung getrennt beobachten und festhalten.

<a name="evidence-available-at-the-time"></a>

## Damals verfügbare Belege

- Der ursprüngliche private Zeitbericht dokumentierte eine abgelaufene Messung vor der Aktivierung. Beide Zehn-Sekunden-Versuche schlugen mit 98,74 und 45,05 Sekunden fehl. Die neue Epoche und die erste Bindung beim zweiten Versuch belegten keine Wiederaufnahme eines zuvor angehaltenen Hilfsprogramms.
- Ein separater privater Bericht zum Normalbetrieb dokumentierte einen funktionierenden Start, eine zweite echte Messung 43,423 Sekunden später, einen verifizierten Kontrollpunkt, den regulären Abschluss und das Prozessende. Dieser Lauf prüfte keine gesetzte SAVE-Sperre.
- Damals konnte `scripts/candidate.py` nach einem gescheiterten `resume` eine neue technische Epoche öffnen und die Steuerdaten wiederherstellen. Dem ersten `activate` oder `bind-active-goal` fehlte dasselbe Journal zur Wiederholung. Für die Implementierung waren eigene Regressionsnachweise nötig.
- Frühere Wartungsarbeiten korrigierten Konflikte bei Freigaben und Anweisungen sowie vier Fehler in Testhilfen und Wartevorgängen. Dieses Verhalten sollte weiter durch Regressionstests abgesichert bleiben.

<a name="requirements"></a>

## Anforderungen

1. Wiederhole einen Erststart unter bestehender Freigabe nur bei protokolliertem `usage_receipt_expired` für dasselbe Ziel und denselben Lauf. Erforderlich sind ein echter neuer Nutzungsbeleg, eine tatsächliche Beobachtung des aktiven Ziels und der Nachweis, dass kein Prozess mehr läuft. Dafür sollen keine neue Nutzeraussage, keine neue Laufidentität und keine umgeschriebenen Nutzungsdaten nötig sein.
2. Verweigere ausgeschöpftes Kontingent, vorzeitige Reserve, einen Nutzerstopp, ein inaktives Ziel, unklare Prozesszugehörigkeit, geändertes Paket oder geänderte Belege und ein anderes Ziel. Python prüft Konsistenz und Format. Es authentifiziert die vom Aufrufer genannte Person oder das Werkzeug nicht unabhängig.
3. Halte fehlgeschlagene Versuche sichtbar. Ein nur teilweise initialisierter Lauf darf nicht aktiv erscheinen. Erhalte spätere tatsächliche SAVE-Belege.
4. Bereite Paket und Test vor der ersten Messung vor. Speichere den tatsächlichen Beobachtungszeitpunkt. Durch einen neueren Schreibvorgang wird eine alte Messung nicht aktuell. Halte `ingest` und die Verlängerung der Laufberechtigung als getrennte Schritte.
5. Im Normalbetrieb sollen echte Entscheidungen `CHECKPOINT_SOON` und `SAVE_AND_STOP` auslösen. Erzwinge dabei keinen künstlichen Stopp.
6. Weise das native Continue und die Wiederaufnahme eines tatsächlich aktiven und danach angehaltenen Workflows getrennt von der Erstbindung nach.
7. Führe die Zeitdiagnose nur auf Anforderung aus. Bereite das Prüfwerkzeug vor dem echten nativen Ereignis vor. Eine verspätete Beobachtung oder zu späte Vorbereitung darf nicht bestehen. Für die Anforderung von SAVE wurden 10 ± 1 Sekunden vorgeschlagen. Dokumentiere die tatsächliche Dauer.
8. Der Hauptagent ruft die nativen Zielwerkzeuge weiter auf dem Host auf. Füge für den Hintergrundprozess keinen undokumentierten Host-Zugang, keinen privaten Codex-Transport und keinen Zugriff auf Anmeldedaten hinzu. Ein exakter Zeitpunkt für einen UI-Klick setzt eine tatsächliche native Ereignisquelle voraus.
9. Prüfe Kontrollpunkt, Prozessende, Abschluss des Hilfsprogramms und Abschluss des nativen Ziels jeweils eigenständig. Pausiere nur mit der Freigabe des Nutzers und verwende `blocked` nur nach den Kriterien des nativen Zielwerkzeugs.
10. Behalte Python 3.11 oder neuer und die Standardbibliothek bei. Ein Windows-Freigabefehler beweist weder Prozessende noch Erfolg. Erhalte bestehende lokale Änderungen und alte Laufzeitbelege. Ändere ihre Bindungen nicht.

<a name="original-authorization"></a>

## Ursprüngliche Freigabe

Dieses Dokument begann als Planung. Der Nutzer erlaubte für jene Arbeit später die Implementierung, Commit und Push des v1.1-Quellcodes und die Überarbeitung der Dokumentation. Ein GitHub-Release und ein Tag lagen außerhalb des Umfangs. Ein echter zeitgesteuerter Stopp blieb separat auszuwählen. Die alten Laufbindungen wurden beibehalten.
