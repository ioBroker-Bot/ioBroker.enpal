# GitHub Copilot Richtlinien für ioBroker.enpal

Diese Datei definiert klare Regeln und Best Practices für Copilot bei der Arbeit am Adapter `ioBroker.enpal`.

## Projektkontext

`ioBroker.enpal` liest Energiedaten aus einer lokalen InfluxDB-2.x-Instanz (Enpal-System) und erzeugt daraus ioBroker-States.

Technische Eckpunkte dieses Repos:
- Sprache: JavaScript (kein Build-Schritt für Laufzeitcode)
- Laufzeit: Node.js >= 22
- Adapter-Core: `@iobroker/adapter-core`
- Admin UI: `admin/jsonConfig.json` (JSON-Config, kein React-Admin)
- Datenquelle: lokal, Polling
- Verbindung: HTTP/HTTPS zur InfluxDB Query API

Wichtige Dateien:
- `main.js`: kompletter Adapter (Polling, Influx-Abfrage, CSV-Parsing, State-Erstellung)
- `io-package.json`: Metadaten, Native-Konfiguration, `info.connection`
- `admin/jsonConfig.json`: Konfigurationsoberfläche
- `admin/i18n/*.json`: Übersetzungen für Admin-Texte
- `.github/workflows/test-and-release.yml`: CI und Release

## Repo-spezifisches Verhalten

### InfluxDB-Anbindung

- Nutze die InfluxDB Query API unter `/api/v2/query?org=<org>`.
- Sende Flux als `POST` mit:
  - `Authorization: Token <token>`
  - `Content-Type: application/vnd.flux`
  - `Accept: application/csv`
- Unterstütze sowohl `http` als auch `https` anhand der konfigurierten URL.
- Bei HTTP-Fehlern `info.connection` auf `false` setzen und sauber loggen.

### Polling und Lebenszyklus

- Polling-Intervall basiert auf `native.interval_s` (Sekunden, Standard 60).
- Polling immer mit `setInterval` starten und beim Unload mit `clearInterval` stoppen.
- In `onReady()` zuerst `info.connection=false` setzen und erst nach erfolgreicher Abfrage auf `true` wechseln.

### State-Modell

- States werden dynamisch aus Influx-Zeilen erzeugt.
- ID-Schema:
  - `enpal.<instance>.<measurement>.<device>.<field>`
  - Wenn kein Device-Tag vorhanden ist: `enpal.<instance>.<measurement>.<field>`
- Zeichen außerhalb `[a-zA-Z0-9_-]` müssen in `_` normalisiert werden.
- Einheiten:
  - `Percent` wird zu `%`
  - `None` wird nicht angezeigt (leere Unit)

## Codequalität

- Bevorzuge einfache, robuste JavaScript-Lösungen.
- Nutze `async/await` für asynchrone Logik.
- Keine unnötigen neuen Abhängigkeiten einführen.
- Kleine, fokussierte Änderungen statt großer Refactorings ohne Not.
- Öffentliche oder komplexe Logik knapp kommentieren, aber keine redundanten Kommentare.

## Linting und Checks

Verwende die vorhandenen Skripte aus `package.json`:
- `npm run lint`
- `npm run lint-fix`
- `npm run check`

Regel:
- Nach jeder relevanten Codeänderung `npm run lint` ausführen.
- Falls möglich zuerst `npm run lint-fix`, danach verbleibende Probleme manuell beheben.

## Tests

Verfügbare Test-Skripte:
- `npm test` (JS-Tests + Pakettests)
- `npm run test:js`
- `npm run test:package`
- `npm run test:integration`

Hinweise:
- Integrationstests laufen über `@iobroker/testing`.
- `main.test.js` ist aktuell ein Platzhalter und sollte bei neuen Features durch echte Tests ergänzt werden.
- Neue Funktionalität möglichst mit mindestens einem sinnvollen Test absichern.

## Admin-UI und Übersetzungen

- Konfigurationsänderungen nur in `admin/jsonConfig.json` umsetzen.
- Zugehörige Texte in den i18n-Dateien konsistent halten.
- Keys zwischen `jsonConfig.json` und i18n müssen exakt übereinstimmen.
- Keine verwaisten oder ungenutzten Übersetzungskeys hinterlassen.

Wenn neue Konfigurationsfelder eingeführt werden:
1. Feld in `admin/jsonConfig.json` anlegen
2. Übersetzungskeys in den relevanten `admin/i18n/*.json` ergänzen
3. Defaults und Validierung in `io-package.json` (`native`) prüfen

## Dokumentation

- `README.md` bleibt auf Englisch (ioBroker-Standard).
- Bei Nutzer-sichtbaren Änderungen README aktualisieren (Konfiguration, Verhalten, Datenpunkte).
- Changelog-Eintrag unter `### **WORK IN PROGRESS**` in `README.md` ergänzen.

Format für Changelog-Einträge:
- `- (author) User-facing Beschreibung`
- Keine fetten Präfixe wie `**FIXED**`, `**NEW**` usw.

## CI/CD Orientierung

Die Pipeline in `.github/workflows/test-and-release.yml` nutzt:
- `ioBroker/testing-action-check@v1`
- `ioBroker/testing-action-adapter@v1`
- `ioBroker/testing-action-deploy@v1`

Wichtige Reihenfolge:
- Erst `check-and-lint`
- Danach `adapter-tests`
- Deploy nur bei Version-Tag

## Klare Arbeitsregeln für Copilot

- Erlaubt:
  - Dateien anpassen
  - Lint/Check/Tests lokal ausführen
  - Fehler beheben und Dokumentation aktualisieren
- Nicht erlaubt:
  - `git commit`, `git push`, Release-Tagging
  - Eigenmächtiges Ausführen von Release-Automation ohne explizite Anweisung

## Enpal-spezifische Implementierungsdetails

Beim Arbeiten in `main.js` besonders beachten:
- `queryInflux()` entscheidet anhand des URL-Protokolls zwischen `node:http` und `node:https`.
- `parseCsv()` muss robust gegen unvollständige/kommentierte CSV-Zeilen bleiben.
- `ensureParentChannels()` erstellt fehlende Channel-Hierarchien vor State-Anlage.
- `syncInfluxToIoBroker()` ist der zentrale Sync-Pfad und darf bei Fehlern nicht abstürzen.
- `show_sync_info` steuert, ob erfolgreiche Syncs als `info` geloggt werden.

Bei Änderungen an Datenpunktnamen oder Parsinglogik immer auf Rückwärtskompatibilität achten, damit bestehende Visualisierungen und Automationen nicht unbeabsichtigt brechen.
