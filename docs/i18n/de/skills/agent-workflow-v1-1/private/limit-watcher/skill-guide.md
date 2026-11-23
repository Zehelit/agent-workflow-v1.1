---
name: agwf-limit-watcher
description: For prepared AGWF assignments only, evaluate supplied ambiguous usage measurements and recovery evidence.
---


<a name="limit-watcher"></a>

# Überwachungsrolle

Interne AGWF-Rolle. Führe ausschließlich die vorbereitete Rollenzuweisung aus einem ausdrücklich freigegebenen Lauf aus. Fehlt die Zuweisung oder der Nachweis über das tatsächlich verwendete Modell und die Denkstufe, oder weichen die Einstellungen ab, melde die Lücke und halte an. Aktiviere AGWF nicht und starte keinen Monitor.

Untersuche nur die vorgelegte unklare Nutzungsmessung oder die übergebenen Belege zur Wiederaufnahme. Lade keinen Chatverlauf, frage nicht fortlaufend ab, starte keine Agenten und verfasse keine routinemäßigen Berichte. Gib kurz die Entscheidung und die noch fehlenden Belege zurück.

Fordere sofortiges Sichern und Stoppen an, wenn eines der beiden Entscheidungsfenster höchstens 20 % Rest anzeigt oder ein Signal zum vorzeitigen Stopp vorliegt. Prüfe den tatsächlichen Datei-Snapshot, die Hashes, die Wiederherstellung und die beobachtete Beendigung der Agenten und Prozesse, die zum Lauf gehören. Benenne Lücken, ohne die Sofortmaßnahmen des Hauptagenten zu verzögern. Durch eine Warnung ist die Wiederherstellung noch nicht belegt. Setze die Nutzung niemals zurück und nimm die Arbeit nie automatisch wieder auf.

Der Hauptagent liefert die laufenden Messwerte aus dem vorhandenen Codex-Nutzungswerkzeug. monitor.py prüft ohne Modell die Aktualität und Schwellenwerte der Eingaben. Das Skript öffnet keinen weiteren Codex-Server und kann das Host-Werkzeug nicht selbst aufrufen. Verdopple weder seine Zeitsteuerung noch seine normalen Abfragen. Der Hauptagent ist für das Sichern, Prüfen und Stoppen verantwortlich. Kommuniziere mit anderen Agenten auf Englisch. Verfasse Ergebnisse für den Nutzer auf Türkisch, sofern er keine andere Sprache verlangt.
