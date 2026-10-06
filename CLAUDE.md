# Leitstand

Persönliches Projekt-Cockpit von Patrick, unabhängig von Ascensus — Status,
Stand und nächste Schritte über alle laufenden Projekte an einem Ort.
Details zur Architektur: `README.md`.

## Wenn eine Sitzung gebeten wird, "im Leitstand nachzuschauen"

Das heißt: in PocketBase (`https://pb.ascensus.fit`) die Collection
`projekte` lesen. Authentifizierung über die Auth-Collection `api_clients`
(dasselbe Konto, das auch im `ascensus-homepage`-Repo für `rezepte` & Co.
genutzt wird) — entweder über ein in der Umgebung hinterlegtes Secret
(E-Mail/Passwort oder Token als Umgebungsvariable, Name variiert, `env`
prüfen) oder, falls keins gesetzt ist, den Nutzer danach fragen statt ein
Passwort zu raten oder einzufordern.

Felder pro Datensatz: `name`, `status` (`offen` / `in_arbeit` /
`wartet_auf_freigabe` / `erledigt` / `pausiert`), `stand`, `schritte`
(beide: ein Punkt pro Zeile, `\n`-getrennt), `quelle`, `erstellt`,
`aktualisiert`, `claude_auftrag` (Zeitstempel, nullable).

**Konvention für Aufträge:** Ein Projekt mit gesetztem `claude_auftrag`
ist bewusst an Claude übergeben worden — der Nutzer hat im Leitstand auf
"→ An Claude senden" geklickt. `stand` beschreibt den Kontext, `schritte`
enthält oft direkt eine Anweisung als letzter Punkt (z. B. "bitte X
erledigen"). Diese Projekte zuerst behandeln.

Wird stattdessen ein konkreter Projektname genannt ("Projekt Test"), direkt
danach filtern (`filter=name="..."`), unabhängig von `claude_auftrag`.

**Vorgehen:**
1. Datensatz(e) lesen, `stand` und `schritte` genau lesen — die enthalten
   oft eine eingebettete Anweisung, keine bloße Notiz.
2. Den Auftrag tatsächlich bearbeiten (nicht nur bestätigen, dass er
   angekommen ist).
3. Ergebnis zurückschreiben: i. d. R. ein neuer Punkt in `schritte`
   (ans Ende anhängen, `\n`-getrennt, bestehende Punkte nicht löschen),
   bei Bedarf `status` aktualisieren. `claude_auftrag` bleibt stehen (der
   Nutzer löscht die Markierung im Leitstand selbst über "Markierung
   entfernen"), außer er bittet ausdrücklich darum.
4. Niemals `aktualisiert` von Hand setzen auf einen Wert in der
   Vergangenheit — aktuelle Zeit beim Schreiben.

API-Beispiel (api_clients-Token `$TOKEN`):
```
curl -H "Authorization: $TOKEN" \
  "https://pb.ascensus.fit/api/collections/projekte/records?filter=claude_auftrag!=''"
```
