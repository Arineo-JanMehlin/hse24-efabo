# HSE24 EFABO – Erkannte Probleme & Risiken

> Sammeldokument aus der Solution-Analyse (Export EFABO v1.0.1.5 unmanaged, DEV, 07.08.2026).
> Details in `Analyse/01-flows.md`, `Analyse/02-canvas-app.md`, `Analyse/03-berechtigungen-solution.md`.
> Kundenseitig gemeldete Bugs B1–B4: siehe UMSETZUNGSLEITFADEN.md Abschnitt 4.

## Offene Punkte aus Anforderungsphase

| ID | Problem | Quelle | Nächster Schritt |
|----|---------|--------|------------------|
| P1 | Unmanaged Layer auf Prod (Flow-Referenz in App als Hotfix entfernt/neu hinzugefügt). DEV/Prod-Drift möglich – DEVTEST-Flow hinkt z. B. inhaltlich hinter Prod-Variante her (ITSec-Case fehlt). | Notiz 05.08. + Analyse | Vor GoLive: Prod-Solution-Layer prüfen (Make-Portal → Solution Layers) und entfernen; im DEV-Export **nicht** sichtbar → Prod-Zugriff nötig. |
| P2 | Unklar, ob automatisierte Mail an definierte Adresse im Angebot mitgeschätzt. | E-Mail Brunner 06.08. | Intern klären (Robert Mueller), Nachtrag per Mail an HSE zur Freigabe. |
| P3 | Flows ohne Fehlerhandling (bis auf CheckifuserinSharePointGroup hat **kein** Flow Failed-Pfade). | Analyse | Beim Neubau: Try/Catch-Scopes + Fehler-Response an App. |

## Befunde aus Solution-Analyse

Schwere: 🔴 funktionaler Defekt · 🟠 Risiko/instabil · 🟡 Unsauberkeit/Ballast

| ID | Befund | Komponente | Schwere | Fix geplant? |
|----|--------|------------|---------|--------------|
| F1 | Druck-Buttons ohne Fehlerbehandlung: `Notify(cvarMessage, Success)` – bei Flow-Fehler leere Notify-Zeile. **= Ursache Bug B1.** | App: btn_PrintAF/btn_PrintITSec | 🔴 | Ja – neuer Button (#2) mit IfError |
| F2 | `btn_PrintAF.Visible`-OR-Kette defekt: nur erster Vergleich gültig, Rest nackte Strings → zuverlässig nur bei „Abgeschlossen" sichtbar. | App: AF-Screen Z. 4267 | 🔴 | Ja – entfällt mit neuem Button |
| F3 | CreateHTML_PDF_AF holt ITSec-Item nur für PDF-Titel → **Flow scheitert, wenn kein ITSec-Item existiert** (zweite B1-Ursache); PDF heißt fälschlich „IT Security Fragen". | Flow CreateHTML_PDF_AF | 🔴 | Ja – im konsolidierten Flow (#2) |
| F4 | ITSec-Druck-CSS: zweiter th-Block `min-width:150px` bei bestehendem `max-width:100px` → **max < min** = Überlappungen/kaputte Umbrüche. AF-CSS: feste px-Breiten, kein `width:100%`, kein `table-layout:fixed`, keine `page-break`-Regeln. **= Ursache Drucklayout-Beschwerde.** | Flows CreateHTML_PDF_* | 🔴 | Ja – #1 Drucklayout |
| F5 | **B2-Ursache:** ITSec-/Einsatz-Software-Berechtigungslogik existiert nur im Create-Flow „Neuer Antrag". „[Child] EFABO Entwurf zu Prüfung" hat keinen ITSec-Block; „Geänderter Vertrag" feuert bei Feld-Update nicht und setzt keine Berechtigungen. | Flows | 🔴 | Ja – ITSec-Block im Child EzP ergänzen |
| F6 | Parameter-Mapping Parent→Child „Berechtigungen ändern" **gekreuzt** (Role↔NewUserMail) → nachträgliche Genehmiger-/Verantwortlichen-Wechsel setzen keine korrekten Berechtigungen (getbyemail('Genehmiger') schlägt fehl). | Flow [Parent] Berechtigungen ändern | 🔴 | Ja – 2-Zeilen-Fix im Mapping |
| F7 | `btn_APVerlängern_1` auf ITSec-Screen wirkungslos (kein Control konsumiert `cvarAPRenew` dort). **Plausible B4-Ursache.** | App: ITSec-Screen Z. 181 | 🟠 | Ja – Button entfernen |
| F8 | Rollenermittlung case-sensitiv: nur eine Seite `Lower()` (App.OnStart Z. 233/240) → `varRolleUser="Antragssteller"` scheitert je nach E-Mail-Schreibweise. | App.OnStart | 🟠 | Ja – beidseitig Lower() |
| F9 | **B3-Verdacht:** `SubmitForm` + direkt folgendes `Patch(…Attachment.Updates)` auf dasselbe Item = Race/ETag-Konflikt; Hilfsformulare ohne `Item`; kein IfError. | App: Edit/View/AF/ITSec-Screens | 🟠 | Ja – Reihenfolge + IfError |
| F10 | Kern-Flows „Neuer Antrag", „Geänderter Vertrag" (+DEVTEST) im Export **deaktiviert** (StateCode 0). In DEV prüfen; Import in Zielumgebung liefert Automatiken deaktiviert. | Solution | 🟠 | Prüfen + nach Import aktivieren |
| F11 | „Geänderter Vertrag - DEV TEST": Duplikat mit identischer Trigger-Condition (**Doppel-Feuer-Risiko**), inhaltlich veraltet (ITSec-Ablehnungs-Case fehlt). | Flow DEVTEST | 🟠 | Löschkandidat (Rückfrage) |
| F12 | „Test FLow Anlage": Testartefakt, hartkodierte ID 174, ungenutzt, **aktiviert**. | Flow | 🟡 | Löschkandidat (Rückfrage) |
| F13 | „Element archivieren": archiviert nur AF-, keine ITSec-Items (Lücke seit ITSec-Erweiterung); Delete nach Create ohne Fehlerhandling/Rollback. | Flow | 🟠 | Empfehlung: ITSec-Archivierung nachrüsten (außerhalb Angebot → P2-Klärung) |
| F14 | Teams-groupId/channelId in „Geänderter Vertrag" hartkodiert; `hse_TeamsID*`-Env-Vars tot. Read-RoleDefId 1073741826 hartkodiert. | Flow/Env-Vars | 🟡 | Optional bei Umbau #3 mitziehen |
| F15 | Mail-Bugs: leere ID im Betreff (Neuer Antrag Z. 2793); doppelt HTML-encodierter Body (Child EzP); „Ende der Leistungserbringung" prüft falsches Feld (AF-HTML); doppelt verwendetes Feld `Andere_Softwareloesungen_Zugriff` (ITSec-HTML). | Flows | 🟡 | Ja – bei #1/#3 mitfixen |
| F16 | App-Fehlerbehandlung kaputt: `ViewForm.OnFailure` referenziert `EditForm.Error` (anderer Screen); `AF_Form.OnFailure` komplett stumm; `NewForm.OnFailure`-Detail auskommentiert; `btn_EntwurfAbschicken` setzt `NextStatus` nie (nackte Record-Literale ohne UpdateContext); Statusvergleich mit eingebettetem Zeilenumbruch matcht nie. | App | 🟠 | Ja – bei App-Anpassung mitfixen |
| F17 | Hartkodierte Testuser in App.OnStart (robert.mueller_external, jan.mehlin_external, desvc.efabo, lisa.grueneisl) in GenehmigerColl. | App.OnStart | 🟡 | Rückfrage: vor Prod entfernen? |
| F18 | Ballast: TestingScreen (unerreichbar, abweichende Rollenlogik-Kopie), Screen2 (leer), AvailableColorsAndControlsScreen, btn_TEST, tote Icons (Icon.Clock auf Druck-Position), unbenutzte Datenquellen (Archiv, Document Library). | App | 🟡 | Optional Cleanup |
| F19 | Druck-Flows laufen `runtimeSource: invoker` (App-User-Kontext, PDF in dessen OneDrive). Für Auto-Druck (#3) zwingend auf embedded/Service-Account umstellen. | Flows CreateHTML_PDF_* | 🟠 | Ja – Teil #2/#3 Design |
| F20 | `hse_EFABOTesting` Default `true` (Mail-Präfix „TEST: "); `hse_EFABO_Document_Library` reist mit Wert im Paket. | Env-Vars | 🟡 | Deployment-Checkliste |
