<a name="command-contract-a02"></a>

# Befehlsspezifikation (A02)

Das mit der Standardbibliothek implementierte Hilfsprogramm prüft Befehle, die über `Workflow.execute` und `Workflow.apply` laufen. Es fängt keine nativen Werkzeuge ab, authentifiziert weder Werkzeugbelege noch Nutzernachrichten, stoppt keine Prozesse und beobachtet effektive Modelle nicht selbst. Der Hauptagent führt diese Schritte aus und liefert Belege. Die Freigaberegeln aus A26 und die Kontrollen für Kontrollpunkte, Zugehörigkeit und Kapazität bleiben in Kraft.

Für den Zustand gilt der ganzzahlige Wert `schema_version: 2`. Zustand ohne Version, aus früheren oder späteren Versionen oder mit falschem Format wird vor jeder Änderung abgelehnt. Es gibt kein Werkzeug für Import oder Migration alter Zustände. Beginne einen neuen AGWF-Lauf mit aktueller typisierter Freigabe. Kontrollpunkte können weiterhin erstellt und wiederhergestellt werden. Historische Belege autorisieren keine Arbeit. Tests mit gezielt ausgelösten Fehlern verwenden ausschließlich Wegwerfkopien.

<a name="json-and-error-policy"></a>

## JSON und Fehlerregeln

Zustandsdateien, `--input`-Dateien, Standardeingabe, Kontrollpunkt-Metadaten und Eigentums-JSON des Installers verwenden denselben strikten Parser. Doppelte Schlüssel werden in beliebiger Verschachtelung abgelehnt, auch bei identischen Werten. `NaN`, `Infinity`, `-Infinity` und endlich aussehende Zahlenliterale, die ins Unendliche überlaufen, etwa `1e999`, werden verweigert. Eingabedateien dürfen UTF-8 mit oder ohne BOM enthalten. Der JSON-Wurzelwert eines Befehls muss ein Objekt sein. Leere Standardeingabe liefert `{}`. Das ist nur bei dafür vorgesehenen Befehlen zulässig. `write_json` validiert und serialisiert mit `allow_nan=False`, bevor ein Verzeichnis oder eine temporäre Datei angelegt wird.

Laufzeiteingaben sind JSON-Objekte mit Zeichenketten als Schlüsseln. Listen bleiben Listen. Ganz- und Gleitkommazahlen sind keine booleschen Werte. Jedes übergebene Flag braucht genau den Python-Typ `bool`, in JSON `true` oder `false`. Zeichenketten, 0/1, null, Listen und Objekte sind dafür nicht erlaubt. Pflichtflags dürfen nicht fehlen. Optionale Flags sind nur bei Auslassung standardmäßig false. Null ist ungültig. Zahlen müssen endlich sein. Anzahlen sind ganze Zahlen ab 1. Zeitstempel sind Ganz- oder Gleitkommazahlen zwischen 0 und sys.float_info.max. Prozentwerte liegen zwischen 0 und 100 einschließlich. Die Zeitstempelgrenze schließt extrem große Ganzzahlen aus, die bei Zeitberechnungen überlaufen würden. Außerdem gelten die Aktualitätsregeln des jeweiligen Befehls. Es gibt kein neues Kapazitätsmaximum. Bei der Zuweisung zählen weiterhin der Hauptagent zum Platzlimit und die bestehende Grenze für Kindagenten.

Eingaben werden vor dem Anlegen der Sperre oder des Verzeichnisses und vor der Befehlsausführung geprüft. `apply` arbeitet auf einer privaten JSON-Kopie und übernimmt sie erst nach bestandenen Kontrollen. Abgelehnte Befehle bewahren Bytes und SHA-256 des Zustands. Ausnahmen sind der unten beschriebene dauerhafte INPUT_INVALIDATED-Beleg und die contract_invalid-Flags. Ein abgelehnter Erstbefehl erzeugt keinen Zustand. Nach einem fehlgeschlagenen Snapshot kann wie bisher ein noch nicht freigegebenes Kontrollpunktverzeichnis des Laufs übrig bleiben. Eine gültig übermittelte, aber nicht verfügbare Nutzung führt absichtlich zum unten beschriebenen Zustandswechsel.

Bei einem Fehler beendet sich das Hilfsprogramm mit Code 1 und gibt dieses Objekt auf der Fehlerausgabe aus.

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| Code | Bedeutung | Feld |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | Befehl des Kerns unbekannt | `$.command` |
| `FIELD_REQUIRED` | Erforderliches Eingabefeld oder Werkzeugoption fehlt | Genaues fehlendes Feld |
| `FIELD_UNKNOWN` | Unbekanntes Eingabefeld oder für den Befehl unzulässige CLI-Option | Genaues beanstandetes Feld |
| `FIELD_TYPE` | Falscher Typ vom Aufrufer | Genaues Feld oder Listenindex |
| `FIELD_VALUE` | Ungültige Identität, leere Pflichtzeichenkette oder -liste, ungültiger Zahlenbereich oder Sitzungsstatus | Genaues Feld oder Listenindex |
| `JSON_DUPLICATE_KEY` | Doppelte Eigenschaft im JSON | Genaues Duplikat, auch in verschachtelten Objekten |
| `JSON_NONFINITE` | Nicht endliche Zahl oder entsprechendes Token, auch in Rohmetadaten | Genaues Feld oder Listenindex |
| `JSON_SYNTAX` | Ungültige JSON-Syntax oder -Kodierung | `$` |
| `JSON_TYPE` | Laufzeitwert außerhalb von JSON oder Objektschlüssel ohne Zeichenkettentyp | Genauer Wert oder Objekt |
| `COMMAND_REFERENCE` | Referenzierte Aufgabe oder erforderliches gespeichertes Feld existiert nicht | `$.id` als Suchkontext. Der Fehlertext nennt den fehlenden Schlüssel. |
| `COMMAND_CONSTRAINT` | Bestehende Prüfung von Freigabe, Zustand, Zugehörigkeit, Überprüfung oder Aktualität | Wenn verfügbar das betroffene Befehlsfeld, bei Prüfung des gesamten Zustands `$.command` |
| `STATE_VERSION` oder `STATE_TYPE` | Nicht unterstützte Zustandsversion oder Wurzelwert kein Objekt | `$.schema_version` oder `$` |
| `IO_ERROR` oder `INVALID_VALUE` | Dateisystemfehler oder anderer Wertfehler | `$` |

Pfade verwenden Punkte für einfache Schlüssel, Klammernotation für andere Schlüssel und `[index]` für Listen. Übliche argparse-Fehler wie unbekannte CLI-Optionen oder Befehle und fehlende Optionsargumente behalten dessen Fehlerausgabe und Code 2. Erkannte, für den Befehl aber unzulässige Optionen erzeugen den strukturierten `FIELD_UNKNOWN`-Fehler. Beide Wege funktionieren ohne `assert`, also auch mit `python -O`.

<a name="payload-types"></a>

## Eingabetypen

`text` ist eine Zeichenkette, die für die Prüfung auch nach Entfernen des äußeren Leerraums nicht leer ist. Der Originalwert bleibt erhalten. `string` darf leer sein. `task_id` erfüllt `[a-z][a-z0-9_]{0,63}`. `strings` ist eine möglicherweise leere Liste von nicht leeren Zeichenketten. `criteria` braucht mindestens einen Eintrag. `task_ids` ist eine Liste von Aufgaben-IDs. `timestamp`, `count` und `percent` folgen den obigen Zahlenregeln. Nur Typen mit `?` erlauben ausdrückliches null.

Jeder unten beschriebene Befehl und jedes strukturierte verschachtelte Objekt verweigert Felder außerhalb seiner Pflicht- und Optionsliste. Zwei Ausnahmen bewahren unverarbeitete Anbieterdaten. Beleginhalte und nicht zur Messung gehörende Metadaten in rohen `rateLimits` oder Zeitfenstern sind JSON, dessen Inhalt nicht interpretiert wird. Doppelte Schlüssel und nicht endliche oder nicht darstellbare JSON-Werte werden auch dort verweigert. Unbekannte Metadaten gelten weder als Befehlsflag noch als gültiger Messnachweis.

| Verschachtelter Typ | Pflichtfelder | Optionen und Einschränkungen |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id` (text), `scope` (scope), `observed_at` und `source_created_at` (timestamp), `original_turn_id` und `observed_turn_id` (text) | Keine weiteren Felder. A26 prüft zulässige Ereignisse, direkte Nutzerquelle, gleichen Umfang, Wiederverwendung und Alter. |
| `scope` | `kind`, `id`, `objective` (text) | `kind` ist objective oder report nach A26. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id` (text), `source_created_at` (timestamp) | Muss mit den Quellenfeldern der übergebenen Freigabe übereinstimmen. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective` (text), `scope` (scope), `goal_created_at` und `observed_at` (timestamp) | Nur für die Erstbindung in N02. Umfang vom Typ objective mit exakt demselben Zieltext. Ursprüngliche Zielerstellung vor der Beobachtung. Keine Felder für menschliche Quellen. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id` (text), `observed_at` (timestamp) | `objective` (text) und `goal_created_at` (timestamp) optional, bei nativer Grundlage aber immer erforderlich und identisch zur Bindung, auch bei Aktualisierungen. A26 verlangt get_goal-Herkunft, passende Identität, höchstens 60 Sekunden Alter und den passenden aktiven Status. |
| `inventory` | Liste von Objekten mit `id` und `status` (text) | Keine zusätzlichen Felder je Eintrag. Eindeutige IDs, Statuswerte running, interrupting, unknown, idle, stopped und done. Beobachtungsalter 0 bis 60 Sekunden. |
| `models` | Objekt, das nicht leere Modellnamen auf Listen `strings` mit Denkstufen abbildet | Leere Objekte und Listen sind zulässig. prepare braucht trotzdem die genaue beobachtete Kombination aus Modell und Denkstufe. |
| `arguments` | Keine strukturellen Pflichtfelder | Nur `task_name` (task_id), `fork_turns` (text), `message` (string) sowie `model` und `reasoning_effort` (text). Record verlangt vollständige Übereinstimmung mit vorbereiteten Argumenten. Fehlende Felder verhindern deshalb die Zuweisung. |
| `receipt` | Nicht leeres Objekt, nicht leere Liste oder Zeichenkette | Gesamter Inhalt aus dem tatsächlichen Werkzeugergebnis als unangetastetes JSON. Keine Echtheitsaussage. |
| `rate_limits?` | Objekt oder null | primary und secondary optional (window?). Weitere Rohmetadaten des Anbieters werden nicht interpretiert. |
| `window?` | Objekt oder null | `usedPercent` (percent?), `windowDurationMins` (count?) und `resetsAt` (timestamp?) optional. Weitere Metadaten werden nicht interpretiert. |

<a name="engine-commands"></a>

## Befehle des Kerns

Die Listen sind vollständig. Gibt es kein optionales Feld, akzeptiert die Eingabe genau die Pflichtfelder. Befehle ohne Felder erhalten `{}`.

| Befehl | Pflichtfelder | Optionale Felder und Standards |
| --- | --- | --- |
| `activate` | `objective` (text), `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). Fehlt es oder ist es null, gilt keine zusätzliche Grenze für offene Sitzungen. |
| `bind-active-goal` | `objective` (text), `native_binding` (native_binding), `interpretation` und `procedural_check` (text), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). Nur im exakt ursprünglichen inaktiven schema2-Zustand ohne Aufgaben, weitere Felder oder Historie. |
| `observe-authorization` | `authorization` (authorization), `interpretation` (string), `source_message` (source) | `app_goal_observation` (app_goal) wird als Ansicht akzeptiert. Der Befehl aktualisiert die gespeicherte Zielbeobachtung nicht. |
| `app-goal` | `app_goal_observation` (app_goal) | Keine |
| `usage` | Keine | `observed_at` (timestamp), `rateLimits` (rate_limits?). Auslassung steht für eine nicht verfügbare Messung. |
| `usage-due` | Keine | `substantial_work` (bool, false) |
| `inventory` | `inventory` (inventory), `observed_at` (timestamp) | Keine |
| `task` | `id` (task_id), `role` (text), `objective` (text), `criteria` (criteria) | `file_inputs` (file_inputs, []), `context_refs` (strings, []), `review_targets` (review_targets, []), `revision` (text, sonst erzeugte UUID), `writes` (strings, []), `depends_on` (task_ids, []), `independent_review` (bool, false) |
| `prepare` | `id` (task_id) | `effort` (text, bei Auslassung Rollenstandard), `current_context` (context_pin für N06) |
| `record` | `id` (task_id), `submitted_arguments` (arguments), `session_id` (text), `receipt` (receipt) | `effective_model` und `effective_effort` (text?). Auslassung oder null bedeutet ungeprüft. |
| `dispatch_failed` | `id` (task_id), `evidence` (text) | `certain_not_started` (bool, false). Fehlt es oder ist es false, bleibt der Ausgang unklar. |
| `reconcile` | `id` (task_id), `evidence` (text), `no_owned_work_running` (bool) | Erfordert true und einen aktuellen Sitzungsbestand ohne aktive Kindagenten. |
| `finish` | `id` (task_id), `accepted` (bool), `evidence` (text) | `review_task` (task_id), bei Abnahme mit Pflicht zur unabhängigen Prüfung erforderlich. |
| `cancel` | `id` (task_id), `reason` (text) | Nur für eine bereite Aufgabe, die nie zugewiesen wurde. |
| `stop` | Keine | Keine |
| `snapshot` | `writers_quiescent` (bool), `next_action` (text) | `files` (strings, []). Das Pflichtflag muss true sein. |
| `restore` | `destination` (text) | Neues Ziel außerhalb des Projekts. |
| `halt` | `inventory` (inventory), `observed_at` (timestamp), `owned_processes_stopped` (bool), `stop_evidence` (text) | `degraded` (bool, false). Bei true sind `reason` und `next_action` (text) erforderlich. Das Prozessflag muss true sein. |
| `resume` | `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal) | Keine. Erfordert passende Fortsetzung nach dem endgültigen Anhalten, frische Nutzung und einen aktuellen Sitzungsbestand und einen verifizierten aktuellen Kontrollpunkt. |
| `compact` | Keine | Keine. Alle Aufgaben beendet und angenommen oder abgelehnt, keine aktiven Kindagenten und aktueller verifizierter Kontrollpunkt. |
| `complete` | Keine | Keine. Kontrollen wie bei compact plus aktiver Lauf. |
| `status` | Keine | Keine. Ein bereits aktivierter Zustand muss vorliegen. |

Alle Dateizuständigkeiten verwenden weiterhin genaue projektbezogene Pfade, abgesichert gegen Windows-Aliasse, Grenzüberschreitungen und symbolische Links. Rollen und Denkstufen folgen dem vorhandenen Katalog und den Verfügbarkeitsprüfungen. Leere Schreiblisten sind für schreibgeschützte Prüf- und Koordinationsrollen möglich. Unbekannte Aufgabenreferenzen schlagen fehl. Aufgaben werden nicht implizit angelegt. `accepted: false` weist eine Aufgabe bei beendetem Schreibzugriff zurück. `accepted: true` nimmt sie erst an, wenn die geltenden Prüfbedingungen erfüllt sind.

<a name="raw-usage-lifecycle"></a>

## Verarbeitung roher Nutzungsdaten

N06 `context_pin` besteht aus genau `{path, sha256, manifest_path, manifest_sha256}`. Verwende nicht leere projektbezogene Pfade und SHA-256-Werte in Kleinbuchstaben. Das Manifest muss als unveränderliche Dateieingabe der Aufgabe vorliegen. `prepare` verifiziert eine aktuelle version1-Ansicht gegen den gegenwärtigen Zustand und das separat abgesicherte Manifest. Mit einer Zusammenfassung vorbereitete Nachrichten sind an dieses Verfahren gebunden. `dispatch-start` nimmt zusätzlich `current_context` entgegen und verlangt eine neu erzeugte Zusammenfassung des Zustands nach prepare mit dem ursprünglichen Manifest. Alle nativen Kontrollen gelten weiter. Siehe [Kontext](context.md) für das Ausgabewerkzeug, Hindernisse aus Belegen, die Prüfung des Erstellers und die Grenzen.

N07 `status` behält die bisherige Tabelle der Aufgabenzustände und ergänzt `task_observations: {contract_version: 1, tasks: ...}`. Native Ausführung und Abnahme des Artefakts sind getrennt. Eine Aufgabe mit done bleibt ausstehend, bis die Kontrollen von finish erfolgreich sind. Unabhängige Abnahme erfasst den vorhandenen optionalen Aufgabenlink `review` und verlangt zwei nicht leere und verschiedene native Sitzungsidentitäten. Zuweisung und Aufzeichnung müssen bei den geprüften Ausgabe-Hashes übereinstimmen. schema2 ändert sich nicht. Eine Prüfung darf weder direkt noch mittelbar von der Abnahme ihres eigenen festgelegten Erstellers abhängen. Historische Abnahmen ohne vollständige Prüfer- und Erstelleridentitäten oder Prüf-Hashes erscheinen als `historical_unrecorded`. Sie erfüllen keine neue Abnahmeabhängigkeit bei prepare, record oder dispatch-start. Die Zusammenfassung und das Hilfsprogramm verwenden dasselbe Unabhängigkeitskriterium. Historie wird nicht automatisch migriert.

Falsche Typen oder Zahlenbereiche führen zu Fehlern und ändern die Zustandsbytes nicht. Das betrifft etwa `usedPercent: "10"`, boolesche Anzahlen oder Zeitstempel, 101 %, negative Rücksetzzeiten, Listen für Fenster oder nicht endliche Metadaten. Fehlendes oder auf null gesetztes `rateLimits`, fehlende oder auf null gesetzte Fenster und Messwerte, typgerechte Messungen ohne eines der Fenster für 300 oder 10080 Minuten und sonst gültige, aber alte oder zukünftige Beobachtungen stehen für nicht verfügbare Nutzung. Ein so gültiger `usage`-Befehl entfernt alte Nutzung absichtlich und erhöht `usage_errors`. Beim ersten Mal folgt `RETRY_USAGE_ONCE`. Beim zweiten Mal folgt `SAVE_AND_STOP` mit Stoppzustand, sofern der Lauf nicht bereits endgültig angehalten oder abgeschlossen wurde. Ein angehaltener Lauf wird dadurch nie still aktiviert. 0, 100 und endliche Bruchteile in Prozent sind gültig. Bei 100 greift die Wiederherstellung für den Schwellenwert. Fehlerhaft formatierte Nutzung erhöht den Zähler für unbekannte Messungen nicht. Rohe Metadaten bleiben erhalten, können aber keine erforderlichen gültigen Fenster ersetzen.

<a name="supported-pinned-entry"></a>

## Unterstützter abgesicherter Einstiegspunkt

Alle Vorgänge verwenden `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`. Laufmodi verlangen zusätzlich `--project <absolute-project>` und `--run-id <fresh-run>`.

| Modus | Eingabe | Verhalten |
| --- | --- | --- |
| `workflow validate` | Keine Eingabe | Prüft freigegebenes Paket und Rollenregeln und bindet einen inaktiven Namensraum |
| `workflow <typed-command>` | `--input <json-file>` oder Standardeingabe | Führt die oben beschriebenen typisierten Befehle aus |
| `coordination` | `--input <json-file>` | Führt eine begrenzte Reihe von Schritten ohne Modell aus. Keine automatische native Zuweisung. |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | Verwendet den internen, abgesicherten Monitor und den laufzugehörigen Zustand |
| `install` | Ausdrückliches `--destination .../agent-workflow-v1-1`, optional `--update` | Prüft Paketdateien und Zugehörigkeitsregeln der Installation |

Der direkte CLI-Aufruf von `workflow.py` wird verweigert. Das Programm stellt keinen Einstieg für Installation, Export oder native Rollenerzeugung bereit. Erzeuge Auslieferungsdateien mit `tooling/build-candidate.py` und installiere in Codex nur über den abgesicherten Kandidaten-Installer. Historische Schemakonvertierung ist kein Kandidatenbefehl und darf eine eingestellte Installation nie wieder in Betrieb nehmen.

<a name="input-and-state-boundary"></a>

## Grenze zwischen Eingabe und Zustand

Richtig formatierte boolesche Befehle ergeben bei einer Wiederholung im Wegwerfprojekt dieselben Entscheidungen. Optionale sichere false-Standardwerte bleiben möglich. Typwidrige Werte werden auch dann abgelehnt, wenn sie als wahr interpretiert werden könnten. Der strikte Zustandsparser prüft JSON-Syntax und endliche Zahlen. Freigaben als Zeichenkette und alter Zustand werden abgelehnt. Tests verwenden nur Wegwerfprojekte.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## Typisierte Zuweisung und zeitliche Reihenfolge der Quellen (A04 / GI01)

N02 `bind-active-goal` bindet nach längerer Vorbereitung das aktuelle aktive native Ziel, ohne eine Nutzerquelle zu erfinden oder ihren Zeitstempel zu ändern. Dazu muss der Zustand fehlen oder exakt `{schema_version: 2, status: inactive, tasks: {}}` lauten, mit den üblichen Anführungszeichen um echte JSON-Zeichenketten. Ein bereits migriertes, abgeschlossenes, gestopptes, angehaltenes, benutztes oder sonst nicht ursprüngliches Hilfsprogramm kann keine Erstbindung vornehmen. Die erste Beobachtung ist ein echter `get_goal`-Beleg im Alter von 0 bis 60 Sekunden. Ziel- und Chat-ID, Zieltext, ursprüngliche Erstellung und Beobachtungszeit müssen mit der Bindung übereinstimmen. Die native Zielerstellung darf beliebig alt sein, aber nicht nach der Beobachtung liegen. Zieltext des Befehls und des Zielumfangs müssen gleich sein. Berichtsumfang und erweiterter Umfang werden abgelehnt. Getrennte, nicht leere Belege für Interpretation und Verfahrensprüfung dokumentieren, dass der Hauptagent aktuellen Kontext und neuere Einschränkungen kontrolliert hat. Das Hilfsprogramm kann diese Prüfung nicht authentifizieren.

Der Zustand speichert `native_binding` als aktuelle Grundlage und hält `native_binding_events` und `native_binding_ids` vom menschlichen Freigaberegister getrennt. Jedes Ereignis enthält `binding`, `interpretation`, `procedural_check` und die ursprüngliche `app_goal_observation`. Die Bindungs-ID wird gegen Wiederverwendung als spätere menschliche Ereignis-ID reserviert. Lauf- und Umfangs-IDs bleiben mit der vorhandenen Historie reserviert. Die gebundene Grundlage erlaubt Arbeit nur mit frischer Zielbeobachtung, deren Text und Erstellungszeit übereinstimmen. Menschliche Freigaben behalten ihr exaktes Quellformat und die 300-Sekunden-Regel für die ursprüngliche Quelle. Eine Fortsetzung durch den Nutzer ersetzt die aktuelle native Grundlage, erhält aber die getrennte native Historie. Das gilt auch für eine spätere menschliche Aktivierung. Fehlerhafte, fehlende, doppelte oder widersprüchliche native Einträge oder Reservierungen führen zur Ablehnung, auch nach menschlicher Fortsetzung. Stopp, Snapshot und endgültiges Anhalten gelten weiterhin. Die neue Beobachtung eines aktiven Ziels setzt ein angehaltenes Hilfsprogramm nie automatisch fort.

Schema 2 verweigert das alte Aufgabenfeld `inputs`. Eine neue Validierung darf historische Zeichenketten ausdrücklich als `context_refs` erhalten. Keine davon wird als Datei- oder Aufgabenreferenz interpretiert. `file_inputs` enthält eine Liste exakter `{path, revision}`-Objekte. Bei der Zuweisung werden projektrelative Dateipfade normalisiert. Duplikate bei ignorierter Groß- und Kleinschreibung, Pfade außerhalb der Grenze, reservierte Pfade und Links werden verweigert. `sha256` und `present` werden festgelegt. null und false kennzeichnen eine fehlende Datei. Unveränderliche Dateieingaben dürfen keine Schreibdateien derselben Aufgabe sein. Deren separate Ausgangshashes vor der Zuweisung bleiben erhalten. Eine Aufgabe hat eine ausdrückliche Revision und einen Zuweisungs-Hash für die gesamte Vereinbarung. Wer Eingabe, Revision oder Hashes austauscht und still neu hasht, kann keine alte Arbeit annehmen.

`review_targets` ist eine ausschließlich für Prüfer erlaubte Liste genauer `{producer_id, revision}`-Objekte. Die Aufgabe jedes Erstellers muss genau in dieser Revision existieren. Bei Zuweisung und Prüfung muss ihr Sitzungsstatus frisch als idle, done oder stopped beobachtet sein. Die Zuweisung legt die genauen Ausgabe-Hashes und die Existenz der Dateien fest. Kontextreferenzen erlauben keine Prüfung. Vorbereitete Argumente enthalten sämtliche typisierten Hash-Referenzen. Dateieingaben, Verträge und Ergebnisse der Ersteller werden bei prepare, record, finish des Prüfers und Abnahme der Erstelleraufgabe erneut geprüft. Angenommene Abhängigkeiten werden ebenfalls nochmals geprüft. Für A05 bleibt die Zustandslogik von accepted und running erhalten. Eine unabhängig zu prüfende Erstelleraufgabe kann während der Prüfung running bleiben, obwohl kein aktiver Schreibzugriff vorliegt.

Bei einer veralteten Aufgabenvereinbarung gibt das Programm `INPUT_INVALIDATED` und einen `invalidation`-Beleg aus. Dieser enthält task_id, Zuweisungsrevision, Grund, wenn vorhanden den genauen geänderten Pfad, erwarteten und tatsächlichen Hash oder Revision, at und sortierte affected_task_ids. Verbraucher desselben Pfads sind nur dann Ausgangspunkte der Ungültigkeit, wenn ihre eigenen Hashes, die festgelegte Dateiexistenz oder die Aufgabenvereinbarung nicht zum beobachteten Wert passen. Eine intakte neuere Revision, die auf die aktuellen Bytes oder deren aktuelles Fehlen festgelegt ist, bleibt nach Ungültigkeit einer älteren Referenz bestehen. Ungültigkeit folgt danach den tatsächlichen Abhängigkeits- und Prüfbeziehungen. Kontextzeichenketten oder gleiche Pfade schaffen für sich keine Beziehung. Die operativen Zustände bleiben gleich. `contract_invalid: true` und der gespeicherte Beleg verhindern die Abnahme alter Ausgaben und Prüfungen. Die Wiederherstellung alter Bytes reaktiviert keine frühere Erstelleraufgabe oder angenommene Prüfung. Eine Wiederholung der Kontrolle bewahrt Beleg und Zustandsbytes.

Bei Beobachtung von Eingaben, Prüfausgaben und Abhängigkeitsausgaben werden symbolische Links, Verzeichnisverknüpfungen, Hardlinks, durch Verzeichnisse ersetzte Dateien und Pfade außerhalb des Bereichs vor der Hash-Berechnung verweigert. Eine unsichere Ausgabe macht Ersteller und betroffene Prüf- und Verbraucheraufgaben vor Rückgabe des Fehlers ungültig. Die Prüfung findet bei Prüfzuweisung, prepare, record, finish und Abnahme der Erstelleraufgabe statt. Der Beleg enthält den erwarteten Hash oder null, den exakten relativen Pfad und eine Zeichenkette `actual`, die mit `unsafe: ` beginnt und den festgelegten Beobachtungsgrund enthält. Für fehlende Dateien steht dort null. Fremde Ein-/Ausgabefehler wie EIO bleiben unverändert sichtbar und werden nicht als Änderung am Aufgabenvertrag eingestuft. Geprüft wird der beobachtete Pfad. Das ergibt keinen atomaren Snapshot des ganzen Dateisystems bei gleichzeitigem externem Ersetzen.

Alle anderen Teiländerungen werden verworfen. Direktes `apply` behält nur die Ungültigkeitseinträge im übergebenen Zustand. execute speichert diese vor der Fehlermeldung atomar. Es gibt keinen Befehl zum Zurücksetzen oder Neu-Hashen. Prüfe den Beleg und weise ausdrücklich eine neue Aufgaben-ID und Revision mit frischen Hash-Referenzen zu. Bei anderen fehlgeschlagenen Kontrollen bleiben die Zustandsbytes wie bisher erhalten.

GI01 verlangt source_created_at und original_turn_id in der Freigabe und ihrer genauen Quellenansicht source_message. observed_at und observed_turn_id beschreiben separat die Beobachtung. Die Quelle muss der Beobachtung vorausgehen und darf bei erstmaliger Annahme höchstens 300 Sekunden alt sein. Für die Fortsetzung muss die Erstellung der Quelle nach Stopp, endgültigem Anhalten oder Abschluss liegen. Der Beobachtungszeitpunkt allein reicht nicht. Die Zeitmarke der zuletzt angenommenen Quelle überdauert Fortsetzung und Neuaktivierung. Erstellungszeiten dürfen nicht zurücklaufen. Gleiche Zeiten verlangen denselben ursprünglichen Turn. Ereignis-IDs und ursprüngliche Quellen-IDs bleiben gegen Wiederverwendung gesperrt, auch bei anderer Art, Beobachtungszeit, anderem Lauf oder Chat und nach Fortsetzung. Beobachtete Ereignisse müssen zum aktuellen Ziel, Lauf und Chat passen. Ein ausdrücklicher Stopp muss auch zum Zielumfang passen. Beobachtungen künftiger Absichten, Regeln und Berichtsanfragen können Belege sichern, aber keine Ausführung oder Wiederaufnahme für das aktuelle Ziel erlauben. Ein Stopp am Nutzungsschwellenwert ist ein mechanischer Stopp des Hilfsprogramms. Er ändert weder die Zielbeobachtung in der App noch schafft er eine Pausenfreigabe des Nutzers.

Diese Felder sind vom Aufrufer gelieferte Belege und authentifizieren keinen Anbieter. Das Hilfsprogramm fängt keine direkten nativen Zielwerkzeuge ab und ruft update_goal nie selbst auf. Ein kontrollierter Regressionstest mit historischem Stopp, neuerer Fortsetzung und Wiederverwendung des Kontexts im selben Turn weist die Ablehnung durch das Hilfsprogramm und eine unveränderte aktive Zielbeobachtung in der App nach. N07 ergänzt die oben beschriebene versionierte Ansicht von Ausführung und Abnahme. Für die Kontext- und Cache-Messung mit einem neuen nativen Kindagenten und für die A06/GI02-Pausenintegration stehen gesonderte Belege aus.
