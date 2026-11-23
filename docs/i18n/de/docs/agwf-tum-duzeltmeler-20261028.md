<a name="agwf-monitoring-follow-up-on-28-october-2026"></a>

# Nachtrag zur AGWF-Überwachung am 28. Oktober 2026

> Historischer Wartungsbericht. Er dokumentiert die Arbeiten nach dem [ersten Wartungsschritt](agwf-duzeltmeler-20261028.md). Daraus ergibt sich keine aktuelle Freigabe zur Aktivierung.

| Feststellung | Damals dokumentierte Änderung und Prüfung |
| --- | --- |
| Sechs Rollen-Skills konnten automatisch ausgewählt werden | Jede Rolle erhielt eine eigene Einstellung `allow_implicit_invocation: false` und wurde auf vorbereitete Aufgaben begrenzt. Die Pakettests erfassten alle sieben Skills. |
| Quellcode, Remote-Repository und Installation unterschieden sich | Die benötigten Werkzeuge kamen ins Paket. Die installierten Dateien wurden mit Manifest und ZIP aus einem sauberen Quell-Commit abgeglichen. |
| Lese- und Schreibzugriffe kollidierten unter Windows | Die betroffenen Komponenten erhielten kompatible Freigaben beim Lesen und angepasstes Ersetzen. Temporäre Windows-Dateien dienten zum Testen von Sperren, Wiederholungen und dem Erhalt alter Inhalte. |
| Für jede Messung startete ein zweiter Codex-Server | Dieser Code wurde entfernt. Der Hauptagent ruft das vorhandene Nutzungswerkzeug des Hosts auf. Python liest den Beleg. Die Messtests unterbinden Prozessstarts. |
| PATH konnte eine andere Codex-CLI auswählen | Die Auswahl einer ausführbaren Datei entfiel aus dem Messpfad. |
| Die Fehlerausgabe wurde verworfen | Der Kindserver wurde entfernt. Messfehler behalten höchstens 20 Einträge mit Diagnosecode, Fehlerklasse und Schritt. Unerwartete Fehler des Python-Prozesses bleiben in seiner lokalen Fehlerausgabe. |
| Zwei Schleifen prüften das Paket jede Sekunde | Das Lebenszeichen beschränkt sich auf Lauf- und Epochenidentität. Die vollständige Prüfung erfolgt bei Befehlen und Messungen. |
| Die Tests verwendeten zuvor einen simulierten Server | Die Tests wurden auf das Nutzungsbelegformat umgestellt. Der Bericht nennt außerdem die Prüfung einer echten Codex-Nutzungsantwort über eine temporäre Datei. |
| Die Rollenregel nannte einen falschen Schwellenwert | Der alte Text `%10` wurde an den tatsächlichen Stoppschwellenwert `%20` angepasst. Ein wegen fehlender Daten verweigerter Start entfernt die eigene Aktivmarkierung. |

Der neue Datenpfad nutzt den ursprünglichen Beobachtungszeitpunkt und die Quellenidentität über `--usage-input`. Der Monitor bindet den Beleg an Lauf und Epoche. Fehlende, zukünftige, abgelaufene, zeitlich zurückliegende oder zu einem anderen Lauf gehörende Messungen werden verweigert. Bei erneuter Veröffentlichung desselben Nutzungsbelegs bleibt dessen Beobachtungszeit gleich. Ohne neue Eingabe wiederholt der Monitor die Prüfung und protokolliert anschließend SAVE_AND_STOP.

Der Hauptagent beschafft vor der Frist neue Beobachtungen. Python ruft das Host-Werkzeug nicht auf. `ingest` veröffentlicht ausschließlich Daten. Aktivierung, Prozessstart, Freigabe, Aktualisierung der Zielbelege und Aufhebung eines Stopps sind getrennte Schritte.

Der Bericht nennt 111 gezielte Tests im Standardmodus und mit `-O`. Tatsächliche Starts von AGWF oder Codex waren dabei unterbunden. Paketbau und Tests für schreibgeschützte Werkzeuge konnten erlaubte Unterprozesse nutzen. Die Quellprüfungen, installierten Hashes, deaktivierten Skill-Einstellungen und die Reproduktion aus einem sauberen Quellstand wurden separat festgehalten.

Laut Bericht wurde auch ein echtes Host-Nutzungsergebnis über den Belegpfad eingelesen, ohne einen weiteren Prozess zu starten oder den Beobachtungszeitpunkt zu ändern. Das Paket erhielt keine Konto- oder Identitätsdaten. Die genaue Ursache der früheren Remote-Störung blieb ungeprüft. Der laufende Parallelbetrieb von AGWF und Remote lag außerhalb des Tests. Siehe [native Nutzungsbelege](../skills/agent-workflow-v1-1/references/native-usage.md).
