# Leitstand

Persönliches Projekt-Cockpit: Status, Stand und nächste Schritte über alle
laufenden Projekte an einem Ort. Reines HTML/CSS/Vanilla-JS, keine
Abhängigkeiten, kein Build-Schritt.

## Architektur

- **Datenhaltung**: PocketBase (`pb.ascensus.fit`), Collection `projekte`
  (`name`, `status`, `stand`, `schritte`, `quelle`, `erstellt`, `aktualisiert`).
  API-Regeln sind 1:1 von der Collection `rezepte` im `ascensus-homepage`-Repo
  übernommen (`@request.auth.collectionName = "api_clients"`).
- **Frontend**: `index.html` spricht PocketBase direkt per `fetch()` an —
  keine Sandbox, kein Mittelsmann. Meldet sich beim ersten Öffnen einmal mit
  den PocketBase-Zugangsdaten (Auth-Collection `api_clients`) an; der
  resultierende Token liegt danach im `localStorage` des Browsers, nicht im
  Quelltext.
- **Aktualisierung**: beim Öffnen neu geladen, danach alle 25 Sekunden
  automatisch neu abgefragt (Polling) — andere Tools, die direkt in
  `projekte` schreiben, erscheinen hier ohne eigenes Zutun.
- **Stand/Nächste Schritte**: je ein Punkt pro Zeile, werden als Aufzählung
  dargestellt — in der Übersicht gekürzt (nächste Schritte, max. 3 Punkte),
  in der Detailansicht eines Projekts vollständig.

## Deployment

Coolify, Build strategy *Static*, `nginx:alpine`, Publish directory `/`.
Kein Build-Schritt — Coolify liefert `index.html` direkt aus dem Repo aus.
Auto-Deploy bei Push auf `main`.

## Zugriffsschutz

Die Seite selbst ist die einzige Hürde: ohne gültige PocketBase-Zugangsdaten
(Collection `api_clients`) lädt sie keine Daten und zeigt nur die
Anmeldemaske.
