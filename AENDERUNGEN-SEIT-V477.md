# Änderungen seit App-Version 477 (24.11.2025)

**Wichtiger Hinweis zur Vollständigkeit:** Dieses Repo (Git) wurde erst am **07.08.2026** angelegt — der Initial-Commit importiert bereits einen Solution-Export als Baseline (bezeichnet als v1.0.1.5). Zwischen Version 477 (24.11.2025) und dem 07.08.2026 liegen **rund 8,5 Monate, die hier nicht erfasst sind**. Falls in diesem Zeitraum direkt in Studio Änderungen gemacht wurden, sind sie in diesem Dokument **nicht** enthalten. Für den vollständigen Abgleich zu Version 477 ist Power Apps Studio → **Versionsverlauf** die einzige verlässliche Quelle.

Diese Datei dokumentiert ausschließlich die Änderungen, die **im Rahmen dieses Projekts (07.08.–18.08.2026)** an Flows und Canvas-App vorgenommen wurden.

---

## Flows (v1.0.2.0 → v1.0.2.1)

| Änderung | Beschreibung |
|---|---|
| Neuer Druck-Flow | Konsolidierter Druck-Flow (ersetzt alte Druck-Logik), Auto-Trigger ergänzt |
| B2-Fix | ITSec-/Einsatz-Software-Berechtigungslogik fehlte in den Child-Flows „[Child] EFABO Entwurf zu Prüfung" und „Geänderter Vertrag" — ergänzt |
| F6-Fix | Parameter-Mapping Parent→Child „Berechtigungen ändern" war gekreuzt (Role↔NewUserMail vertauscht) — korrigiert |
| F4-Fix (Drucklayout) | CSS-Fehler in `CreateHTML_PDF_*`: `min-width` > `max-width` (Überlappung), fehlendes `table-layout:fixed`/`page-break` — korrigiert |
| F15-Feldfix | Doppelt verwendetes Feld `Andere_Softwareloesungen_Zugriff` im ITSec-Druck-HTML — auf korrektes Feld `DL_Zugriff_auf_Software_Daten` umgestellt; zusätzlich Datumsfeld-Fix im AF-Druck-Flow |
| Paket v1.0.2.1 | Reines Flows-Paket (ohne App) in DEV importiert und verifiziert — App-Rollback dabei bewusst vermieden |

## Canvas App (via Coauthoring-MCP, „Teil 1" + „Teil 2" aus ANLEITUNG-CANVAS-APP.md)

### Teil 1 — Umbau Druck-Button
- `btn_PrintEFABO` neu auf `ViewEFABOScreen` ergänzt
- 7 tote Druck-/Verlängerungs-Controls entfernt (alte Einzel-Buttons/Logik, durch neuen Druck-Flow ersetzt)

### Teil 2 — Fehlerbehandlung
| # | Fix | Beschreibung |
|---|---|---|
| 2.1 | Rollenermittlung | `Lower()` fehlte auf einer Seite des Vergleichs in `App.OnStart` (F8) — beidseitig ergänzt |
| 2.2 | ViewForm-Fehler | `ViewForm.OnFailure` zeigte fälschlich `EditForm.Error` (falscher Screen) statt eigenen Fehler — korrigiert |
| 2.3 | AF_Form stumm | `AF_Form.OnFailure` hatte keine Fehlermeldung — `Notify(...)` ergänzt |
| 2.4 | NextStatus | `btn_EntwurfAbschicken` setzte `NextStatus` nie (nackte Record-Literale ohne `UpdateContext`) — `UpdateContext` ergänzt |
| 2.5 | B3-Spezifikation | Minimal-Fix-Spezifikation für Race-Condition SubmitForm+Patch dokumentiert (Umsetzung geplant) |

### 18.08.2026 — Nachträge
| # | Fix | Beschreibung |
|---|---|---|
| — | ITSec-Notify | `ITSecurityFreigabenScreen.OnFailure` hatte kein Notify — analog zu 2.3 ergänzt |
| F21 | ComboBox-Labels leer | Alle Personen-/Choice-Comboboxen (Verantwortlicher Mitarbeiter, Fachbereich, Genehmiger, Techn. Ansprechpartner HSE, Prüfender Rechtsanwalt — 9 Stellen, 4 Screens) zeigten beim Öffnen leere Zeilen. Ursache 1: totes `"Picture"`-Feld in `DisplayFields` (weder `GenehmigerColl` noch SharePoint-Records liefern das Feld) — entfernt. Ursache 2: Studio-Classic-ComboBox übernahm den YAML-Fix nicht automatisch — musste zusätzlich per Studio-Panel „Primärer Text"/„SearchField" manuell neu gesetzt werden. **Bug bestand bereits im v1.0.1.5-Original.** |
| F22 | Vertragsvolumen-Hinweistext dauerhaft sichtbar | Hinweisbox unter „Geplantes Vertragsvolumen" (`DataCard3`/`DataCard3_2`/`DataCard3_3`) hatte nie eine `Visible`-Formel — seit Original immer sichtbar, unabhängig vom Feldinhalt. `Visible` ergänzt, Bedingung identisch zur bestehenden Submit-Button-Logik. |

---

## Betroffene Dateien (Coauthoring-YAML)

- `coauthoring/NewEFABOScreen.pa.yaml`
- `coauthoring/EditEFABOScreen.pa.yaml`
- `coauthoring/ViewEFABOScreen.pa.yaml`
- `coauthoring/ITSecurityFreigabenScreen.pa.yaml`

## Offene Punkte (nicht Teil dieser Änderungen)

- P4: Service-Account-Zugriff weiterhin ungeklärt (Connection-Sharing)
- F17: Hartkodierte Testuser in `GenehmigerColl` — Rückfrage vor Prod-Umstellung
- F18: Ballast-Screens (TestingScreen, Screen2, AvailableColorsAndControlsScreen) — optionales Cleanup
- F19: Druck-Flows laufen `runtimeSource: invoker`, für Auto-Druck (#3) Umstellung auf Service-Account nötig
- Aktuell in Klärung (18.08., nach diesem Dokument): ComboBoxen zeigen in der **echten Play-Version** der DEV-App nichts an, obwohl übrige App dort unauffällig läuft (siehe PROBLEME.md F21-Nachtrag)
