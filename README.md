# Leitstand

Persönliches Projekt-Cockpit: Status, Stand und nächste Schritte über alle
laufenden Projekte an einem Ort. Reines HTML/CSS/Vanilla-JS, keine
Abhängigkeiten, kein Build-Schritt.

## Architektur

- **Datenhaltung**: PocketBase (`pb.ascensus.fit`), zwei Collections:
  - `projekte` (`name`, `status`, `stand`, `quelle`, `erstellt`,
    `aktualisiert`, `claude_auftrag`, `anhaenge`, `geloescht` — Soft-Delete,
    siehe "Papierkorb" unten)
  - `projekt_schritte` (`projekt`-Relation, `text`, `erledigt`, eigene
    `anhaenge`, `erstellt`, `aktualisiert`, `antwort_auf`-Selbstrelation,
    `autor` select `patrick`/`claude`, `position` number) — jeder nächste
    Schritt ein eigener Datensatz statt eines Punkts in einem gemeinsamen
    Textfeld, damit er einzeln abgehakt, bearbeitet und mit eigenem Anhang
    versehen werden kann. `autor` ist nötig, weil Web-UI, MCP-Connector und
    Skill alle über denselben `api_clients`-Account schreiben und
    PocketBase sie sonst nicht unterscheiden könnte. `position` (nur bei
    Top-Level-Schritten) steuert die per Pfeil-Buttons änderbare Reihenfolge
    der Ideen.

  API-Regeln beider Collections sind 1:1 von `rezepte` im
  `ascensus-homepage`-Repo übernommen (`@request.auth.collectionName =
  "api_clients"`).
- **Frontend**: `index.html` spricht PocketBase direkt per `fetch()` an —
  keine Sandbox, kein Mittelsmann. Meldet sich beim ersten Öffnen einmal mit
  den PocketBase-Zugangsdaten (Auth-Collection `api_clients`) an; der
  resultierende Token liegt danach im `localStorage` des Browsers, nicht im
  Quelltext.
- **Aktualisierung**: beim Öffnen neu geladen, danach alle 25 Sekunden
  automatisch neu abgefragt (Polling) — andere Tools, die direkt in
  `projekte` schreiben, erscheinen hier ohne eigenes Zutun.
- **Stand**: ein Punkt pro Zeile in einem Textfeld, als Aufzählung
  dargestellt.
- **Nächste Schritte**: einzelne Datensätze in `projekt_schritte` statt
  Zeilen in einem Textfeld — in der Übersicht gekürzt auf die noch offenen
  (max. 3), in der Detailansicht als Chat-Verlauf pro Gesprächsfaden
  (Patrick links, Claude rechts, `buildChatGroups()`), durchnummeriert
  ("Schritt N", siehe `CLAUDE.md`), mit Abhak-Checkbox, Inline-Bearbeitung,
  Löschen und eigenem Datei-Anhang je Nachricht. Spaltet sich eine Idee in
  mehrere Antworten auf (z. B. mehrere Varianten eines Vorschlags), wird
  daraus je ein eigener, unabhängiger Chat-Block statt einer gemeinsamen,
  themenspringenden Zeitleiste — alle zusammen bilden eine Gruppe, die sich
  per Pfeil-Buttons relativ zu anderen Ideen verschieben lässt.
- **Status-Filter**: die Zählerzeile oben in der Übersicht ist klickbar —
  ein Status anklicken zeigt nur Projekte mit diesem Status, "Projekte"
  setzt den Filter zurück. "Erledigt" ist bewusst klein und ganz rechts
  abgesetzt.
- **Papierkorb**: Löschen eines Projekts ist ein Soft-Delete (`geloescht`
  gesetzt statt Datensatz entfernt). Button oben neben "Abmelden" öffnet
  den Papierkorb, dort einzeln oder alle wiederherstellen oder endgültig
  löschen.

## Deployment

Coolify, Build strategy *Static*, `nginx:alpine`, Publish directory `/`.
Kein Build-Schritt — Coolify liefert `index.html` direkt aus dem Repo aus.
Auto-Deploy bei Push auf `main`.

## Zugriffsschutz

Die Seite selbst ist die einzige Hürde: ohne gültige PocketBase-Zugangsdaten
(Collection `api_clients`) lädt sie keine Daten und zeigt nur die
Anmeldemaske.
