# Leitstand

Persönliches Projekt-Cockpit von Patrick, unabhängig von Ascensus — Status,
Stand und nächste Schritte über alle laufenden Projekte an einem Ort.
Details zur Architektur: `README.md`.

## Wenn eine Sitzung gebeten wird, "im Leitstand nachzuschauen"

Das heißt: in PocketBase (`https://pb.ascensus.fit`) die Collection
`projekte` lesen, plus bei Bedarf die zugehörigen Schritte aus
`projekt_schritte` (siehe unten — getrennte Collection, kein Textfeld
mehr). Authentifizierung über die Auth-Collection `api_clients`
(dasselbe Konto, das auch im `ascensus-homepage`-Repo für `rezepte` & Co.
genutzt wird) — entweder über ein in der Umgebung hinterlegtes Secret
(E-Mail/Passwort oder Token als Umgebungsvariable, Name variiert, `env`
prüfen) oder, falls keins gesetzt ist, den Nutzer danach fragen statt ein
Passwort zu raten oder einzufordern.

### Collection `projekte`

`name`, `status` (`offen` / `in_arbeit` / `wartet_auf_freigabe` /
`erledigt` / `pausiert`), `stand` (ein Punkt pro Zeile, `\n`-getrennt,
weiterhin Fließtext/Aufzählung), `quelle`, `erstellt`, `aktualisiert`,
`claude_auftrag` (Zeitstempel, nullable), `anhaenge` (Dateifeld,
mehrere, allgemein am Projekt).

### Collection `projekt_schritte` (seit 08.10.2026, ersetzt das frühere `schritte`-Textfeld)

Jeder "nächste Schritt" ist ein **eigener Datensatz**, kein Punkt mehr in
einem gemeinsamen Textfeld — Patrick wollte das explizit so, damit ein
neuer Schritt nicht in bestehenden Text reingeschrieben werden muss und
damit ein Anhang gezielt an einen einzelnen Schritt hängen kann statt nur
allgemein ans Projekt.

Felder: `projekt` (Relation auf `projekte`), `text`, `erledigt` (bool),
`anhaenge` (Dateifeld, mehrere, eigene je Schritt), `erstellt`,
`aktualisiert`, `antwort_auf` (Selbst-Relation auf `projekt_schritte`,
nullable, seit 08.10.2026 — siehe unten).

Schritte zu einem Projekt lesen:
```
GET /api/collections/projekt_schritte/records?filter=projekt="<projekt-id>"&sort=erstellt
```

**`antwort_auf` — Antworten gehören in die Box ihrer Idee, nicht daneben:**
Verweist ein Schritt per `antwort_auf` auf einen anderen, zeigt die
Oberfläche ihn eingerückt innerhalb von dessen Box statt als eigene,
gleichrangige Zeile. Wird ein Auftrag aus einem bestehenden Schritt heraus
bearbeitet (z. B. der letzte Eintrag enthält eine Anweisung), gehört das
Ergebnis als Antwort **auf genau diesen Schritt**, nicht als neuer
Top-Level-Schritt daneben — sonst reißt es optisch wieder auseinander,
was inhaltlich zusammengehört (das war explizit der Punkt, den Patrick
nach der ersten Version bemängelt hat).

**Konvention für Aufträge:** Ein Projekt mit gesetztem `claude_auftrag`
ist bewusst an Claude übergeben worden — der Nutzer hat im Leitstand auf
"→ An Claude senden" geklickt. `stand` beschreibt den Kontext, der
**letzte** `projekt_schritte`-Eintrag enthält oft direkt eine Anweisung
(z. B. "bitte X erledigen"), kein bloßes To-do. Diese Projekte zuerst
behandeln.

Wird stattdessen ein konkreter Projektname genannt ("Projekt Test"), direkt
danach filtern (`filter=name="..."`), unabhängig von `claude_auftrag`.

**Vorgehen:**
1. Projekt lesen (`stand`) und seine `projekt_schritte` dazu — besonders
   den letzten/neuesten Eintrag genau lesen, der enthält oft eine
   eingebettete Anweisung, keine bloße Notiz. Verweist ein Schritt auf
   einen Anhang ("siehe Screenshot"), den unter `anhaenge` dieses
   Schritt-Datensatzes (nicht des Projekts) suchen und laden
   (`GET /api/files/projekt_schritte/<schritt-id>/<dateiname>`).
2. Den Auftrag tatsächlich bearbeiten (nicht nur bestätigen, dass er
   angekommen ist).
3. Ergebnis zurückschreiben: einen **neuen** `projekt_schritte`-Datensatz
   anlegen (nicht in einen bestehenden reinschreiben), verknüpft über
   `projekt` mit der Projekt-ID — und, falls die Arbeit eine Antwort auf
   einen konkreten vorhandenen Schritt ist (der Normalfall), zusätzlich
   über `antwort_auf` mit dessen ID, damit das Ergebnis in dessen Box
   erscheint statt lose daneben:
   ```
   POST /api/collections/projekt_schritte/records
   Body: {"projekt": "<projekt-id>", "antwort_auf": "<id des Schritts, auf den geantwortet wird>",
          "text": "...", "erledigt": false,
          "erstellt": "<jetzt, ISO 8601>", "aktualisiert": "<jetzt>"}
   ```
   (`antwort_auf` weglassen, wenn es sich um einen wirklich neuen,
   eigenständigen nächsten Schritt handelt statt um eine Antwort auf
   einen bestehenden.)
   Bei Bedarf `status` des Projekts selbst aktualisieren (PATCH auf
   `projekte/records/<id>`). `claude_auftrag` bleibt stehen (der Nutzer
   löscht die Markierung im Leitstand selbst über "Markierung
   entfernen"), außer er bittet ausdrücklich darum.
4. Niemals `aktualisiert` von Hand setzen auf einen Wert in der
   Vergangenheit — aktuelle Zeit beim Schreiben.

API-Beispiel (api_clients-Token `$TOKEN`):
```
curl -H "Authorization: $TOKEN" \
  "https://pb.ascensus.fit/api/collections/projekte/records?filter=claude_auftrag!=''"
```
