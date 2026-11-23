<a name="repeatable-lifecycle-check"></a>

# Lebenszyklus reproduzierbar prüfen

`tooling/run-lifecycle-check.py` verarbeitet echte Beobachtungen zu Ziel, Nutzung und Sitzungsbestand der Kindagenten über den abgesicherten Kandidaten. Der Hauptagent beschafft die Beobachtungen beim Host. Halte Sitzung und Eingaben privat und außerhalb von Git. Nur ein Agent schreibt die Sitzungsdatei.

<a name="prepare-before-measuring"></a>

## Vor der ersten Messung vorbereiten

Baue den Kandidaten und bewahre den freigegebenen externen Manifest-Hash auf. `prepare` prüft das Paket und erfasst eine neue Sitzung, ohne Kandidat oder Monitor zu starten. Schließe die Vorbereitung vor der ersten tatsächlichen Nutzungsmessung ab.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` stammt aus dem freigegebenen Build-Beleg. Verwende ein bestehendes Projekt und eine neue Sitzungsdatei. Binde nach einer Paketänderung keine alte Sitzung an einen anderen Hash.

<a name="host-input-object"></a>

## Eingabeobjekt des Hosts

`run`, `observe`, `resume`, `close` und `timed-test` akzeptieren ein JSON-Objekt über `--input`.

| Feld | Inhalt |
| --- | --- |
| `activation` | Nur beim ersten `run`. Die vollständige typisierte Kandidateneingabe für `bind-active-goal` mit `native_binding`, passendem `app_goal_observation`, Zieltext, Interpretation, Verfahrensprüfung, beobachteten Modellen und Denkstufen, Kapazitätsgrenze, tatsächlichem Sitzungsbestand und Beobachtungszeit. |
| `app_goal_observation` | Aktuelles `get_goal` mit Ziel- und Chatidentität, Zieltext, Erstellungszeit und Status, dazu `source_kind: get_goal`, ein echter Quellenverweis und `observed_at`. Ergänze den tatsächlichen Wert `updated_at`, falls vorhanden. |
| `usage` | `source`, `sourceId`, `observedAt` und `result` aus den [nativen Nutzungsbelegen](../skills/agent-workflow-v1-1/references/native-usage.md). Übernimm nur die vorgeschriebenen Fenster. |
| `inventory` | Tatsächliche aktuelle Identitäten und Zustände der Kindagenten. Verwende `[]` nur, wenn es keine Kindagenten gibt. |
| `checkpoint_files` | Geänderte, hinzugefügte oder gelöschte projektbezogene Pfade für die Wiederherstellung, einschließlich der Dateien des Hauptagenten. Der Kern ergänzt die Dateien der Aufgaben. |
| `writers_quiescent` | `true` erst nach der Beobachtung, dass alle Agenten das Schreiben beendet haben. Das Prüfwerkzeug kann diese Beobachtung nicht erzeugen. |
| `next_action` | Konkreter nächster Schritt nach Wiederaufnahme oder regulärem Abschluss. |
| `weekly_exception` | Optional beim ersten `run`. Tatsächliche Nutzeraussage `exact_quote` und befristeter Unix-Ablauf `valid_until`. Standardmäßig gilt keine Ausnahme. |
| `resume` | Optional die vollständige Kandidateneingabe für die Wiederaufnahme. Andernfalls versucht das Werkzeug `usage_recovery: true` ausschließlich bei protokolliertem Ablauf der Messung. Kontingent- und Nutzerstopps behalten ihre eigenen Voraussetzungen. |

Liefert die native API keine eigenständige Ziel-ID, verwende eine einheitlich abgeleitete Identität wie `threadId:createdAt` und kennzeichne sie als abgeleitet. Die typisierten Formate beschreiben der [Betriebsleitfaden](../skills/agent-workflow-v1-1/references/operations.md) und die [Befehlsspezifikation](../skills/agent-workflow-v1-1/references/commands.md).

<a name="run-refresh-and-close"></a>

## Ausführen, aktualisieren und abschließen

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Erfasse den tatsächlichen Beobachtungszeitpunkt, sobald der native Aufruf zurückkehrt. Das Werkzeug prüft `status.at == usage.observedAt`, die Identität des laufzugehörigen Prozesses, Epoche, Lebenszeichen, Messwert und Laufberechtigung. `observe` verlängert auch die Laufberechtigung. `refresh_by` schlägt eine frühere Aktualisierung vor und ändert die eigentliche Frist nicht.

Verifiziere bei `CHECKPOINT_SOON` einen Kontrollpunkt für die fertige Arbeit. Halte `checkpoint_files` aktuell. Ein reguläres `close` erfordert aktuelle Überwachung, einen überprüften Snapshot, den Abschluss und das beobachtete Prozessende. Im Normalbetrieb wird kein SAVE-Ereignis künstlich ausgelöst.

Beim ersten `run` mit `weekly_exception` richtet das Werkzeug die befristete Monitorregel ein und meldet `NEED_POLICY_OBSERVATION`, bevor es den Workflow aktiviert. Ermittle neue Ziel- und Nutzungsbeobachtungen und wiederhole danach dieselbe Sitzung. Erhalte den ursprünglichen Zeitstempel jedes früheren Belegs.

<a name="halt-and-resume-after-expiry"></a>

## Nach Ablauf anhalten und wiederaufnehmen

Fordert der Monitor SAVE an, führt `observe` Stopp, Snapshot und endgültiges Anhalten aus und meldet `NEED_POST_HALT_OBSERVATION`. Beschaffe danach neue Beobachtungen zu Ziel, Nutzung und Sitzungsbestand.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

Der Kern verwendet die tatsächlich beobachtete Ursache aus dem Monitor. Die technische Wiederaufnahme verlangt dasselbe freigegebene aktive Ziel, unveränderte Dateien und Kontrollpunkt, neue Nachweise nach dem Stopp und ein geprüftes Ende des laufzugehörigen Prozesses. Für ausgeschöpftes Kontingent, vorzeitige Reserve und ausdrücklichen Nutzerstopp gelten die jeweils passenden Fortsetzungsregeln. Das Werkzeug versetzt das native Ziel nicht in paused, blocked oder active.

Bei einer fehlgeschlagenen Erstaktivierung behält die Sitzung den Hash des Startversuchs des Kandidaten. Verwende denselben `run` mit neuen Belegen nur, wenn die Voraussetzungen für eine Wiederholung erfüllt sind. Eine veränderte Sitzung ist kein gültiger Nachweis. Fertige oder unklare Versuche werden nicht automatisch neu gestartet.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## Zeitdiagnose nur auf ausdrückliche Anforderung

Bereite vor dem Ereignis einen ordnungsgemäß laufenden Prozess vor. Verwende `timed-test` nur für die ausdrücklich angeforderte Diagnose. Das Ereignis-JSON enthält `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` und `observed_at` aus einer echten nativen Beobachtung.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

Ein verspätetes Ereignis führt zu `INVALID_TIMING`. `SAVE_REQUEST_ON_TIME` bedeutet, dass die SAVE-Anforderung innerhalb des zulässigen Zeitfensters lag. Eine native Pause braucht eigene Host-Belege. Das Werkzeug lässt daher `native_pause_verified: false` und `native_paused_at: null` stehen. Ein vollständiger echter Test mit zwei Durchgängen beobachtet erst den aktiven, dann den angehaltenen und nach einer tatsächlichen autorisierten Fortsetzung erneut den aktiven Zustand desselben Ziels und Laufs. Ein Test mit künstlicher Uhr prüft allein die Logik.
