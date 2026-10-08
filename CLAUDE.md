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
mehrere, allgemein am Projekt), `geloescht` (Zeitstempel, nullable, seit
08.10.2026 — Soft-Delete, siehe "Papierkorb" unten).

**Soft-Delete (Papierkorb):** Ein gelöschtes Projekt wird nicht sofort aus
PocketBase entfernt, sondern bekommt `geloescht` gesetzt und verschwindet
damit aus der normalen Übersicht. Im Papierkorb (Button oben neben
"Abmelden") lässt es sich wiederherstellen (`geloescht: null`) oder
endgültig löschen (echtes `DELETE`). Beim Schreiben über die Leitstand-UI
automatisch so — eine Sitzung, die direkt per HTTP ein Projekt entfernen
soll, sollte standardmäßig genauso verfahren (PATCH `geloescht` statt
DELETE), außer Patrick bittet ausdrücklich um endgültiges Löschen.

### Collection `projekt_schritte` (seit 08.10.2026, ersetzt das frühere `schritte`-Textfeld)

Jeder "nächste Schritt" ist ein **eigener Datensatz**, kein Punkt mehr in
einem gemeinsamen Textfeld — Patrick wollte das explizit so, damit ein
neuer Schritt nicht in bestehenden Text reingeschrieben werden muss und
damit ein Anhang gezielt an einen einzelnen Schritt hängen kann statt nur
allgemein ans Projekt.

Felder: `projekt` (Relation auf `projekte`), `text`, `erledigt` (bool),
`anhaenge` (Dateifeld, mehrere, eigene je Schritt), `erstellt`,
`aktualisiert`, `antwort_auf` (Selbst-Relation auf `projekt_schritte`,
nullable, seit 08.10.2026 — siehe unten), `autor` (select `patrick` /
`claude`, required, seit 08.10.2026 — siehe "Chat-Darstellung" unten),
`position` (number, optional, seit 08.10.2026, nur bei Top-Level-Schritten
gesetzt — steuert die Reihenfolge der Ideen, siehe "Ideen verschieben"
unten).

Schritte zu einem Projekt lesen:
```
GET /api/collections/projekt_schritte/records?filter=projekt="<projekt-id>"&sort=erstellt
```

**`antwort_auf` — Antworten gehören zum Gespräch ihrer Idee, nicht daneben:**
Verweist ein Schritt per `antwort_auf` auf einen anderen, zeigt die
Oberfläche ihn als Chat-Nachricht in derselben Unterhaltung statt als
eigene, unabhängige Zeile (Details zur Darstellung unten). Wird ein
Auftrag aus einem bestehenden Schritt heraus bearbeitet (z. B. der letzte
Eintrag enthält eine Anweisung), gehört das Ergebnis als Antwort **auf
genau diesen Schritt**, nicht als neuer Top-Level-Schritt daneben — sonst
reißt es optisch wieder auseinander, was inhaltlich zusammengehört (das
war explizit der Punkt, den Patrick nach der ersten Version bemängelt
hat).

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
          "text": "...", "erledigt": false, "autor": "claude",
          "erstellt": "<jetzt, ISO 8601>", "aktualisiert": "<jetzt>"}
   ```
   `autor` ist Pflichtfeld — bei allem, was eine Sitzung schreibt, immer
   `"claude"` (nie für Patrick raten). Über den `leitstand-mcp`-Connector
   wird es serverseitig fest gesetzt, nur beim rohen HTTP-Weg hier explizit
   mitschicken.
   (`antwort_auf` weglassen, wenn es sich um einen wirklich neuen,
   eigenständigen nächsten Schritt handelt statt um eine Antwort auf
   einen bestehenden.)
   Bei Bedarf `status` des Projekts selbst aktualisieren (PATCH auf
   `projekte/records/<id>`). `claude_auftrag` bleibt stehen (der Nutzer
   löscht die Markierung im Leitstand selbst über "Markierung
   entfernen"), außer er bittet ausdrücklich darum.
4. Niemals `aktualisiert` von Hand setzen auf einen Wert in der
   Vergangenheit — aktuelle Zeit beim Schreiben.

### Chat-Darstellung (seit 08.10.2026, ersetzt die frühere Baum-/Einklapp-Ansicht)

Eine erste Version zeigte Antworten als verschachtelte, eingerückte Boxen.
Bei langen oder tiefen Ketten wurde die Spalte dabei immer schmaler (jede
Ebene fügte erneut Checkbox, Löschen-Button und Abstände hinzu, unabhängig
von der Einrückung selbst) — zwei Zwischenlösungen (Einrücktiefe deckeln,
dann zusätzlich automatisches Einklappen) haben das Grundproblem nur
verschoben. Die jetzige Lösung verzichtet komplett auf Verschachtelung:

- **`buildChatGroups()`** gruppiert zunächst nach Top-Level-Idee (jeder
  `projekt_schritte`-Eintrag ohne `antwort_auf` ist eine eigene Gruppe).
  Innerhalb einer Gruppe zerlegt sie den `antwort_auf`-Baum in flache
  Gesprächs-"Segmente": eine Kette von Schritten, die ununterbrochen genau
  einer auf den anderen antworten. Hat ein Schritt **mehrere** Antworten
  (z. B. eine Idee, aus der mehrere Varianten wurden — "Post 1/2/3"), endet
  das laufende Segment dort, und **jede** Antwort startet ein neues eigenes
  Segment, das mit demselben verzweigenden Schritt als gemeinsamem Kontext
  beginnt. So bleibt jeder Gesprächsfaden für sich lesbar, statt dass
  mehrere unabhängige Themen in einer Chronologie durcheinanderspringen
  (das war Patricks expliziter Punkt gegen eine einzige durchgehende
  Zeitleiste pro Projekt).
- Jedes Segment wird als eigene `.chatbox` gerendert — eine Liste von
  Chat-Blasen (`bubbleHtml()`), **Patrick links, Claude rechts** je nach
  `autor`-Feld (siehe oben) —, darunter ein immer sichtbares
  Antwort-Eingabefeld, das an die letzte Nachricht des Segments anhängt
  (`antwort_auf` = deren ID). Da innerhalb eines Segments nie verschachtelt
  wird, bleibt die Breite unabhängig von der Länge der Konversation
  konstant.
- **Einklappen einer Box** (seit 08.10.2026, zweite Iteration): Jede Box mit
  mehr als einer Nachricht lässt sich per Klick bis auf die Ausgangsfrage
  einklappen — unabhängig vom Alter oder der Länge des Gesprächs. Schlüssel
  dafür ist die **letzte** Nachricht des Segments (`lastId`), nicht die
  erste — die erste teilen sich oft mehrere Boxen derselben Idee (z. B. alle
  drei Post-Varianten beginnen mit derselben Ausgangsidee), mit der ersten
  als Schlüssel klappten früher versehentlich alle gemeinsam auf.

### Schritt-Nummern (seit 08.10.2026)

Jede `.chatbox` zeigt oben "Schritt N" — fortlaufend über **alle** Boxen
eines Projekts hinweg nummeriert (nicht pro Idee neu bei 1), damit Patrick
in einer Unterhaltung eindeutig "Schritt 3" sagen kann, auch wenn mehrere
Varianten derselben Idee im selben Projekt stehen (das war genau der
Auslöser: er wollte sich innerhalb eines Themas wie "social media" auf
einen bestimmten Zweig beziehen können, ohne dass Claude ihn mit einem
Nachbarzweig verwechselt).

**Eine Sitzung, die auf "Schritt N" reagieren soll, muss dieselbe Zahl
berechnen können wie die Oberfläche** — die Nummer steht nirgends direkt
als Feld in PocketBase, sie ergibt sich deterministisch aus:
1. Top-Level-Schritte (ohne `antwort_auf`) nach `position` sortieren
   (aufsteigend; bei fehlender/gleicher `position` nach `erstellt`).
2. Für jeden davon der Reihe nach seinen `antwort_auf`-Baum durchlaufen und
   in flache Segmente zerlegen (siehe `buildChatGroups()` oben — bei einer
   Verzweigung ein Segment pro Antwort, alle mit dem verzweigenden Schritt
   als erster Nachricht).
3. Jedes so entstehende Segment bekommt die nächste fortlaufende Nummer,
   in exakt der Reihenfolge, in der es in Schritt 2 erzeugt wird.

Entspricht 1:1 dem, was `leitstand_projekt_schritte` zurückgibt (sortiert
nach `erstellt`) plus `antwort_auf`/`position` — mit diesen Feldern lässt
sich die Nummer jederzeit nachrechnen, ohne die Web-UI zu öffnen.

### Ideen verschieben (seit 08.10.2026)

Top-Level-Ideen (nicht einzelne Antworten) lassen sich per Pfeil-Buttons
(`moveIdea()`) umsortieren. Steuert `position` (number, optional) auf
`projekt_schritte` — nur bei Top-Level-Schritten gesetzt, bei Antworten
irrelevant. Neue Ideen aus der Web-UI bekommen automatisch
`position: Date.now()` (hängt ans Ende an, ohne vorher nachzufragen); ein
Verschieben nummeriert alle Top-Level-Ideen des Projekts neu durch
(0, 10, 20, …), robust auch wenn alte Datensätze noch keine `position`
hatten. **Wichtig:** Verschieben ändert die Schritt-Nummern aller Boxen ab
der verschobenen Stelle — eine vorher notierte "Schritt 3" kann danach
etwas anderes meinen, im Zweifel neu nachsehen statt sich auf eine alte
Notiz zu verlassen.

API-Beispiel (api_clients-Token `$TOKEN`):
```
curl -H "Authorization: $TOKEN" \
  "https://pb.ascensus.fit/api/collections/projekte/records?filter=claude_auftrag!=''"
```
