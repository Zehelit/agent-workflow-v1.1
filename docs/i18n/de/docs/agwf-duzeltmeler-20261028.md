<a name="agwf-maintenance-on-28-october-2026"></a>

# AGWF-Wartung am 28. Oktober 2026

> Historischer Wartungsbericht. Die folgenden Beobachtungen und Freigaben gehören zu den damaligen Arbeiten. Die aktuelle Bedienung beschreibt das [README](../README.md).

Jeder der sechs Rollen-Skills erhielt eine eigene `agents/openai.yaml` mit `allow_implicit_invocation: false`. Die Rollenbeschreibungen gelten für vorbereitete AGWF-Aufgaben. Eine Rolle hält an, wenn die Zuweisung oder der Nachweis des tatsächlich verwendeten Modells und der Denkstufe fehlt. Die Rollen-Skills starten weder den Workflow noch den Monitor. Aus dem Haupt-Skill wurde eine Formulierung entfernt, nach der eine frühere Projektauswahl gültig geblieben wäre.

Quellcode und Installation wurden beim geteilten Lesezugriff, beim Ersetzen unter Erhalt der Berechtigungen und bei begrenzten Wiederholungen für Windows-Zustandsdateien angeglichen. `candidate.py` verwendet denselben Leser mit geteilter Freigabe für Statusdateien, die sich ändern. Sperrt ein externer Leser die Datei, bleibt die alte Fassung erhalten. Eine dauerhafte Sperre ist weiterhin ein Fehler.

Die Werkzeuge für Kontext und Nutzung wurden in den mitgelieferten Ordner `tools/` verschoben. Dadurch ist die Benutzerinstallation nicht vom Projekt-Checkout abhängig. Bei dieser historischen Prüfung stimmten 41 Dateien in Quellcode, Paket und Installation überein. Die alte Kopie des Kontextwerkzeugs im Stammverzeichnis wurde entfernt. Die Pfade in den Tests wurden angepasst.

Der Bericht nennt 63 Paket- und Werkzeugtests sowie neun Tests zum Windows-Dateizugriff. Die neun Windows-Tests wurden außerdem mit `-O` und installierten Dateien ausgeführt. Sieben Skill-Definitionen und die Datei-Hashes wurden geprüft. Der Pakettest für die sechs fehlenden Rollenregeln schlug vor der Korrektur fehl und bestand danach.

AGWF war während der Wartung deaktiviert. Alle sieben Skill-Einstellungen blieben ausgeschaltet. Die Tests prüften Dateizugriffe des Kerns an temporären Dateien. Den laufenden Parallelbetrieb mit Remote haben sie nicht getestet.

Zu den damals offenen Risiken zählten der Start eines zweiten Codex-Servers bei jeder Messung, die Wahl einer anderen CLI aus PATH, der Verlust der Fehlerausgabe des Kindservers und wiederholte Prüfungen des vollständigen Pakets. Die erste Wartung konnte die Ursache der Remote-Störung nicht belegen. Der [Wartungsnachtrag](agwf-tum-duzeltmeler-20261028.md) beschreibt die späteren Änderungen. Der aktuelle Monitor startet keinen zweiten Codex-Server.
