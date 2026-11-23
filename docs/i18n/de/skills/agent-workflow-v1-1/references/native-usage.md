<a name="native-usage-receipts"></a>

# Native Nutzungsbelege

Verwende dieses Verfahren nur in einem ausdrücklich freigegebenen und aktivierten AGWF-Lauf. Ein aktueller ausdrücklicher Aufruf des Skills verlangt einen Start innerhalb des freigegebenen Umfangs. Ist das Anlegen eines nativen Ziels untersagt, verwende die manuelle Überwachung aus dem [Betriebsleitfaden](operations.md). Ein Paketupdate, ein Gespräch über den Skill oder das Veröffentlichen von Nutzungsdaten aktiviert einen deaktivierten Workflow nicht von selbst.

Der Hauptagent ruft `mcp__codex_app__get_usage_limits` über die bestehende Codex-Verbindung auf. Python öffnet keinen zweiten Anwendungsserver, wählt keine andere CLI, liest keine Anmeldedateien, verbindet sich nicht mit privaten Transporten und stellt keine Modellanfragen. Der Hintergrundprozess überwacht die übergebenen Belege und kontrolliert ihre Aktualität. An wichtigen Arbeitsschritten muss der Hauptagent das Host-Werkzeug aufrufen.

Erstelle aus einem tatsächlichen Werkzeugergebnis eine private JSON-Eingabe.

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

Die Zahlen sind Beispiele. Erhalte den tatsächlichen Beobachtungszeitpunkt, die Belegidentität und die gemessenen Werte. Erfinde keine Werte und ersetze den Beobachtungszeitpunkt nicht durch die Veröffentlichungszeit der Datei. Falls das Ergebnis mehrere Kontingentgruppen enthält, verwende vorrangig `rateLimitsByLimitId.codex`. Eine vorhandene Zuordnung ohne Codex wird verweigert. Es wird nicht still auf eine andere Gruppe ausgewichen. Beide Fensterdauern und beide Rücksetzzeiten sind erforderlich. Kopiere nur die benötigten Nutzungsfelder in diese private Eingabe. Konto-, Profil- und Guthabenfelder werden nicht gebraucht. Die veröffentlichte Datei `watcher/usage-source.json` übernimmt nur die freigegebenen Fensterfelder, Identität und Zeit der Beobachtung und die Bindung an Lauf und Epoche.

`workflow activate --input AUTH.json --usage-input USAGE.json` und `bind-active-goal` prüfen die Freigabe vor jedem Prozessstart. Auch `resume` braucht einen neuen Beleg über `--usage-input`. Der Beleg aus einer alten Epoche ist nicht wiederverwendbar. Der manuelle Start mit `watcher start --usage-input USAGE.json` folgt denselben Vorprüfungen. Bei fehlendem oder bereits abgelaufenem Beleg wird der Start abgelehnt, bevor ein Prozess entsteht.

Veröffentliche eine neue Messung während der Arbeit mit `watcher ingest --usage-input USAGE.json` oder der Option `--usage-input` eines Workflow- oder Koordinationsbefehls. `ingest` aktiviert keinen Workflow, startet keinen Prozess, verlängert keine Freigabe und hebt keine gespeicherte SAVE-Sperre auf. Eine neue Quelldatei löst sofort eine Auswertung aus, ohne die alte Frist abzuwarten. Der Status kann kurz noch die vorherige Auswertung zeigen, während der zugehörige Prozess den neuen Beleg einliest. Vor folgenreichen Zuweisungen muss ein aktueller Status bestätigen, dass der Monitor ordnungsgemäß arbeitet.

Der Hauptagent aktualisiert die tatsächliche Werkzeugmessung vor der adaptiven Frist. Sie beträgt höchstens 300 Sekunden und wird in der Nähe eines Grenzwerts oder bei schnellem Verbrauch kürzer. Der Prozess gibt das erneute Lesen einer alten Datei niemals als neue Messung aus. Kann der Hauptagent keine frischen Daten beschaffen, wiederholt der Prozess einmal die Validierung und setzt dann SAVE_AND_STOP. Ein dokumentierter Stopp mit `usage_receipt_expired` erlaubt anschließend unter demselben bereits freigegebenen und weiterhin aktiven Ziel eine technische Wiederaufnahme über `workflow resume` mit `usage_recovery: true`. Siehe [technische Wiederaufnahme](operations.md#expired-telemetry-recovery-under-existing-authority). Neue Daten alleine heben die Sperre nicht auf. Diese kontrollierte Wiederaufnahme erfordert keine neue Nutzernachricht. Ausgeschöpftes Kontingent und ausdrückliche Nutzerstopps behalten ihre eigenen Voraussetzungen. Aktualisiere an Arbeitsschritten rechtzeitig für den nächsten Werkzeugaufruf. Die Frist ist keine Warteempfehlung. Lange, nicht unterbrechbare Aufrufe begrenzen die Aktualisierung über das Host-Werkzeug. Ein unbeaufsichtigter Hintergrundzugriff auf das native Werkzeug wird nicht zugesichert.

Fehlerhafte Eingaben, falsche Lauf- oder Epochenidentitäten, zukünftige oder abgelaufene Beobachtungen, rückläufige Zeitstempel und verändert erneut eingereichte Belege werden verweigert. Das erneute Veröffentlichen eines identischen Belegs lässt den Zustand unverändert. Wie bei anderen nativen Belegen liefert der Aufrufer die Angaben zu Zeit und Herkunft. Ein formal passendes JSON authentifiziert den Anbieter nicht. Halte Rohbelege privat und behandle keine beliebige Quellenangabe als unabhängigen Nachweis.

Messfehler zeigen im Messwert und Status einen erlaubten `diagnosticCode` und eine Fehlerklasse. `watcher/diagnostics.json` hält höchstens 20 strukturierte Einträge. Ausgenommen sind Ausnahmetexte, Anmeldedaten und beliebige Werkzeuginhalte. Unerwartete Fehler des Python-Prozesses stehen weiterhin lokal in `worker-stderr.log`. Fehlerausgaben eines Kindservers können nicht mehr verloren gehen, weil dieser Server nicht mehr gestartet wird.

Jeder Kandidatenbefehl und jede Auswertung des Prozesses, auch von neu veröffentlichten Eingaben, prüft das vollständige Paket. Lebenszeichen kontrollieren nur Lauf, Epoche und zugehörige Steuerdaten und melden den Status. Sie durchlaufen und hashen das Paket nicht erneut. Eine veränderte Abhängigkeit wird bei der nächsten Messung oder beim nächsten Befehl verweigert. Nicht jedes Lebenszeichen erkennt sie.

Die Unit-Tests decken das tatsächliche Schreiben und Lesen von Belegdateien, striktes JSON, Ablauf, Replay-Schutz, Identität, Schwellenwerte und begrenzte Diagnosen ab. Sie prüfen auch, dass der Messpfad keinen Prozess startet. Die Prüfung des laufenden Remote-Produkts erfordert eigene Beobachtungen. Lass den installierten Workflow deaktiviert, bis der Nutzer ihn ausdrücklich wieder aktiviert. Ein Wartungslauf soll keinen echten AGWF-Prozess starten, um diese Abnahme zu fingieren.
