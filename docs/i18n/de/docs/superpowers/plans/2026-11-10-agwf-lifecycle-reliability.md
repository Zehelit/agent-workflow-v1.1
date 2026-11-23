<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# Implementierungsplan für Start und Wiederaufnahme

> Historischer Implementierungsplan vom 10. November 2026. Die Prüflisten halten die damaligen Vorschläge fest. Die Freigaben und erwähnten privaten Artefakte gehören zu diesen Arbeiten. Das aktuelle Verhalten beschreibt der [Betriebsleitfaden](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="goal-and-constraints"></a>

## Ziel und Rahmenbedingungen

Der Plan sah eine sichere Wiederholung des Erststarts, rechtzeitig aktualisierte Messungen, eine verifizierte Wiederaufnahme und einen regulären Abschluss vor. Er behielt den abgesicherten Kandidaten, den typisierten Workflow-Kern und den Monitor ohne Modell bei. Das Prüfwerkzeug sollte Vorbereitung, tatsächliche Host-Beobachtung, Betrieb und Abschluss auseinanderhalten. Der Zeitversuch blieb ein ausdrücklicher Modus. Siehe die [historische Spezifikation](../specs/2026-11-10-agwf-lifecycle-reliability.md).

Der ursprüngliche Plan erlaubte den zugehörigen Quellcode-Commit und Push für v1.1. Ein GitHub-Release und ein Release-Tag waren ausgeschlossen. Er setzte Python ab 3.11, die Standardbibliothek mit unittest, vorhandene Codex-Werkzeuge für Ziele und Nutzung sowie Windows-Prozess- und Dateifreigabeverhalten voraus. Laufende Arbeit, alte Laufzeitbelege und Paketbindungen sollten erhalten bleiben. Bestehendes Paket, Ziel, Messung, Nutzeraussagen und Prozessidentität mussten tatsächlich beobachtet werden. Synthetische Daten gehörten in isolierte Tests. Für Ablauf des Belegs, reales Kontingent, vorzeitige Reserve und Nutzerstopp galten unterschiedliche Freigaben. Der Normalbetrieb sollte kein künstliches SAVE auslösen. Arbeit anderer Sitzungen und Modelleinstellungen des Nutzers waren ausgenommen.

<a name="review-focus"></a>

## Schwerpunkte der Prüfung

1. Ein echtes SAVE wegen abgelaufener Nutzungsmessung nach einem gescheiterten Erststart soll eine zulässige Wiederholung ermöglichen, auch wenn der Zustand des Hilfsprogramms noch nicht existiert.
2. Ein Stopp wegen Kontingent oder Nutzerwunsch, ein neues Ziel oder ein unbekannter Prozess darf die technische Wiederaufnahmeprüfung nicht bestehen.
3. Ein unvollständiger oder paralleler Start bleibt unklar, bis belastbare Belege den Zustand klären.
4. Eine neuere Belegdatei macht ihre Beobachtung nicht neuer. Auch bei aktueller Messung kann die Laufberechtigung abgelaufen sein.
5. Ein verspätetes natives Ereignis kann die Zeitprüfung nicht bestehen. Vorübergehende Windows-Freigabefehler belegen kein Prozessende.

<a name="files-and-responsibilities"></a>

## Dateien und Zuständigkeiten

| Pfad | Geplante Zuständigkeit |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Startversuch, Entscheidung zur Wiederholung, Monitorzugehörigkeit und Ergebnisse des Lebenszyklus |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | Verhalten von SAVE, Epoche und Belegen, nur mit den erforderlichen Änderungen |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Bestehende Regeln für aktiv, angehalten und abgeschlossen sowie den Nutzerstopp |
| `tooling/run-lifecycle-check.py` | Vorbereitetes Prüfwerkzeug mit nativen Belegen des Hauptagenten |
| `tooling/tests/test_lifecycle_check.py` | Tests für Vorbereitung, verspätete Ereignisse und Einstufung der Ergebnisse |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Integration von Quelle und installiertem Paket mit echten laufzugehörigen Python-Prozessen |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` und `test_watcher_atomic_write.py` | Skill-Tests für Beleggrenzen, Ziele und Freigaben, Monitorfristen und Windows-Verhalten |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` und `references/coexistence.md` | Anweisungen für Aktualisierung, Wiederholung, Installation und getrennte Host-Abnahme |

<a name="task-1-retry-a-failed-first-activation"></a>

## Aufgabe 1. Eine gescheiterte Erstaktivierung wiederholen

Geplant waren die Schnittstellen `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` und `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. Pro Lauf erfasste `activation-attempt.json` Quellenidentität, `schema_version`, `command`, `pre_state_sha256`, `phase` sowie die Beobachtungen des Monitors und die Fehlerdetails. `prepared`, `failed` und `committed` sind verschiedene Phasen. Ein nicht abgeschlossener Eintrag mit `prepared` muss mit dem tatsächlichen Zustand abgeglichen werden.

- Weise zuerst das Scheitern mit einem alten Beleg nach. Prüfe dann die Wiederholung mit frischem Beleg und demselben Ziel für `bind-active-goal` und `activate`. Bewahre den ursprünglichen SAVE-Hash und die Nutzerfreigabe. Verifiziere den funktionierenden Prozess und den aktiven Zustand.
- Speichere nach `preflight_activation` die Startabsicht dauerhaft. Binde Befehl, Lauf, Ziel, Chat, Umfang, Zieltext und Freigabe. Eine neuere Beobachtungszeit allein schafft keinen neuen Umfang.
- Verwende zwischen den Schritten von Launcher und Kern einen laufzugehörigen Lebenszyklus-Lock für einen einzelnen Schreiber. Verweigere einen unbekannten Lock-Eigentümer. Native Benutzeroberfläche und Python-Zustand werden getrennt geändert.
- Nutze die neue technische Epoche nur für bestätigtes `usage_receipt_expired` im selben Lauf, eine neue Beobachtung des aktiven Ziels und einen realen Beleg. Ein vom Aufrufer gesetztes Wiederholungsflag erteilt keine Freigabe. Erfasse den Erfolg nach der Prüfung des Prozesses und dem Speichern des Kernzustands.
- Prüfe Negativfälle für Kontingent, vorzeitige Reserve, eine andere Diagnose, ein neueres SAVE, Nutzerstopp, geändertes oder inaktives Ziel, laufende oder unbekannte Prozesse, ein verändertes oder unvollständiges Journal, falsches Paket und parallelen Start. Prüfe die Wiederaufnahme nach einem ungültigen Beleg für den neuen Versuch. Das Rückgängigmachen stellt verifizierte alte Steuerdaten nur im Bereich des betreffenden Fehlers wieder her und erhält jedes neuere SAVE.
- Führe Tests zum Kandidaten-Parallelbetrieb und zu nativen Belegen aus. Prüfe aktiven Zustand, aktuelle Identität und Aktualität, bewahrte Historie und das tatsächliche Beenden des Prozesses.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## Aufgabe 2. Vor der Messung vorbereiten und früh aktualisieren

Das Prüfwerkzeug sollte `prepare`, `run --input PATH`, `observe --input PATH` und `close` anbieten. Die Vorbereitung prüft Paket, Hash, Rollen und Befehle vor dem Start. `refresh_by(at: float, interval: float) -> float` schlägt `at + interval - min(15, interval / 4)` vor. Bei 60 Sekunden Intervall ergibt das eine Aktualisierung nach 45 Sekunden. Die eigentliche Frist bleibt gleich.

- Prüfe, dass prepare und die Hilfe keine echten Eingaben brauchen, alte oder fehlende Eingaben keinen aktiven Zustand schaffen und das erneute Einreichen eines Belegs dessen Beobachtungszeit nicht ändert.
- Ermittle nach der Vorbereitung das native Ziel und die Nutzung. Kopiere ausschließlich freigegebene Felder, darunter beide tatsächlichen Fenster und Rücksetzzeitpunkte. Lasse Konto-, Identitäts- und Guthabendaten weg.
- Unterscheide `sample_at`, `next_check_at`, `check_interval_seconds` und `refresh_by`. Eine erfolgreiche Veröffentlichung oder ein Lebenszeichen beweist keine aktuelle, ordnungsgemäße Überwachung.
- Verlange `status.at == usage.observedAt`, die Identität des laufenden Prozesses, die passende Epoche, `sampleFresh` und `leaseValid`. Prüfe die Verlängerung nach dem Einlesen. Versprich keine Aktualisierung während eines nicht unterbrechbaren Aufrufs.
- Löse in normaler Arbeit keinen künstlichen Timer aus. Prüfe Snapshots bei entsprechender Anweisung und stelle beim regulären Abschluss sowohl den abgeschlossenen Zustand als auch das Prozessende fest. Prüfe, dass dabei keine künstliche SAVE-Sperre entstand.
- Führe die Lebenszyklus- und Belegtests mit unterschiedlichen Beobachtungs- und Veröffentlichungszeiten durch.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## Aufgabe 3. Zeiten prüfen und verspätete Ereignisse als solche ausweisen

- Führe `timed-test` nur auf ausdrückliche Anforderung aus. Verwende einen vorbereiteten funktionsfähigen Prozess, ein echtes Ereignis und `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`. Geplant war `event_at + 10` als Frist. Die Wartezeit wird mit einer monotonen Uhr gemessen.
- Ein um 45 Sekunden verspätet beobachtetes Ereignis soll `INVALID_TIMING` ergeben. Die historischen Versuche mit 98,74 und 45,05 Sekunden bleiben Fehlschläge.
- Nutze bestehende offizielle Zielbelege. Fehlt eine unterstützte Ereignisschnittstelle, stufe die Beobachtung als verspätet oder unvollständig ein. Verwende keine privaten Transportschnittstellen oder Zugangsdaten, um die Benutzeroberfläche zu messen.
- Speichere `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` und `native_paused_at` separat. Das Fenster von 9 bis 11 Sekunden gilt für die Anforderung von SAVE bei zuvor funktionierendem Prozess. Eine native Pause braucht einen eigenen Beleg.
- Verwende in beiden tatsächlichen Durchgängen dasselbe Ziel und denselben Lauf. Belege die aktive Ausführung, den Kontrollpunkt und den erfolgten Stopp, dann eine echte zulässige Wiederaufnahme. Eine erstmalige Bindung eines unbenutzten Zustands belegt keine Wiederaufnahme.
- Teste mit frühen, späten, erneut eingereichten und veränderten synthetischen Ereignissen, fehlendem Pausenbeleg, ausdrücklichem Stopp und beiden Durchgängen. Die Live-Zeitmessung bleibt freiwillig und getrennt von der normalen Abnahme.

<a name="task-4-verify-recovery-and-regressions"></a>

## Aufgabe 4. Wiederaufnahme und Regressionen prüfen

- Prüfe Erststart, Ablauf während des ersten Versuchs und frische Wiederholung, Ablauf eines aktiven Laufs mit Stopp, Snapshot, endgültigem Anhalten und Wiederaufnahme, einen ungültigen und dann einen gültigen Wiederholungsversuch, native Fortsetzung von paused zu active und normalen Abschluss. Teste die jeweils vorgesehenen Einstiege aus Quelle und Paket.
- Beobachte vor weiterer Arbeit `halted` und verifiziere den Snapshot. Prüfe nach der Wiederaufnahme ursprüngliches Ziel und Lauf, neue Epoche, erhaltenen SAVE-Hash, Freigabe und tatsächliche Funktionsfähigkeit des Prozesses.
- Verweigere Kontingent, vorzeitige Reserve, ein neueres Quoten-SAVE, Nutzerstopp, falsches Ziel, veraltete oder wiederverwendete Belege, beschädigte Kontrollpunkte sowie nicht geklärte laufende Kindagenten. Für den nativen Status `blocked` gelten weiterhin eigene Kriterien.
- Kopiere unter Windows nur notwendige Dateien aus dem eigenen Lauf und prüfe bekannte Prozessidentität und Endzustand. Kurzzeitige Freigabefehler bedeuten kein Prozessende. Dauerhafte Rechteprobleme müssen in begrenzter Zeit sichtbar werden.
- Führe nach den gezielten Fällen beide Testsammlungen des Repositorys aus. Ermittle die Testzahlen bei der Ausführung. Für den normalen Live-Zyklus brauchst du ein echtes Ziel und ein vorbereitetes Werkzeug. Ablauf und Zeitdiagnose brauchen eigene Freigaben und dürfen `not_tested` bleiben.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## Aufgabe 5. Regeln dokumentieren und Auslieferung überprüfen

- Dokumentiere die erste Wiederholung, die Wiederaufnahme nach endgültigem Anhalten und die eigenen Abläufe für Kontingent- und Nutzerstopps. Erkläre die Reihenfolge bei Messwertaktualisierung und Verlängerung der Laufberechtigung sowie die Aussagekraft eines verspäteten Ereignisses.
- Erfasse das Paket und die vorhandene Arbeit. Erhalte Änderungen und genutzte Installationen. Leite keinen aktiven Zustand aus der Zugehörigkeit eines alten Hashes ab und migriere keine aktiven Läufe.
- Baue zweimal aus identischer Quelle und vergleiche Manifest- und ZIP-Hashes. Prüfe geänderte Paketdateien beim Bau und in Installationstests. Prüfe den installierten Kandidaten mit einem passiven Wegwerflauf, kontrolliere Hash und Windows-Zugriff und dokumentiere den passiven Umfang.
- Belege `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` und `package_match` einzeln. Erhalte alte negative Ergebnisse.
- Prüfe Freigabe, Wiederherstellung und Erhalt der SAVE-Belege vor einer damals autorisierten Quellcode-Veröffentlichung. Private Berichte, Belege und Laufzeitzustände bleiben außerhalb von Git.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## Aufgabe 6. Vorhandene v1.1-Identität und Präsentation angleichen

- Stelle Anzeigename, Aufrufname, Paketpfad und neuen Namensraum gemeinsam um. Erhalte frühere Laufzeitidentitäten und akzeptierte Hashes.
- Halte die tatsächliche Geschichte von v1.0 und die Änderungen in v1.1 in `CHANGELOG.md` fest. Erkläre im README Zweck, Grenzen, Architektur, Arbeitsablauf und Orchestrierung.
- Beschreibe die Installation von einem sauberen Checkout für Menschen und Agenten einschließlich des gesicherten Hashes, der Überprüfung und des ausdrücklichen Aufrufs. Die Einrichtung startet kein natives Ziel und ändert keine Modelleinstellungen.
- Gleiche für die damals freigegebene Veröffentlichung About und Topics mit dem Quellstand ab. Erhalte Repository-Zugriffsregeln und Branch-Schutz. Prüfe README-Befehle und die Übereinstimmung der Diagramme mit der Quelle.
- Veröffentliche nur freigegebene Dateien, prüfe Remote-Commit und -Dateibaum, Version und Dokumentation. Verifiziere, dass kein Release und kein Tag erstellt wurde.

<a name="acceptance-boundary"></a>

## Abnahmeumfang

Der ursprüngliche Plan ordnete die Kernaufgaben 1 bis 5 vor den abschließenden Veröffentlichungsarbeiten an. Er verlangte Prüfungen der ersten Wiederholung und der Wiederaufnahme nach echtem Anhalten, Regressionstests, Paketprüfungen und echte Beobachtungen mit nachvollziehbarem Umfang. Für das Verhalten der nativen Oberfläche waren echte Host-Nachweise nötig. Das Zehn-Sekunden-Ergebnis war eine gesonderte Aussage, die ein tatsächliches natives Ereignis und eine separat beobachtete Zeitmessung verlangte.
