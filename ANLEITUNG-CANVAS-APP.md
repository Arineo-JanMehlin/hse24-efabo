# Anleitung: Canvas-App-Anpassungen EFABO (Studio)

> Stand 11.08.2026. Voraussetzung: Solution EFABO **v1.0.2.0** ist in DEV importiert (✅ erledigt).
>
> ✅ **11.08.: Teil 1 (bis auf 1.1) und Teil 2 (2.1–2.4) via Coauthoring-MCP umgesetzt** (Commits `1c0a8ae`, `55dd8d6`):
> btn_PrintEFABO auf ViewEFABOScreen (OnSelect = Platzhalter-Notify, echter Flow-Call auskommentiert bis P4 gelöst),
> alle 7 Alt-Controls gelöscht, Bugfixes 2.1–2.4 aktiv. **Noch offen: In Studio speichern + veröffentlichen!**
> Danach offen: 1.1 (Flow als Datenquelle, braucht Teil 0), btn_PrintEFABO-OnSelect scharf schalten, Smoke-Test, 2.5 (B3-Minimal-Fix, **fest eingeplant**, nach Smoke-Test).
>
> ✅ **12.08.: F15-Feldfix im neuen Druck-Flow + Flows-only-Paket gebaut** (siehe Teil 0). Der konsolidierte Flow hatte den Duplikat-Feld-Bug aus `CreateHTML_PDF_ITSec` geerbt: Zeile „Haben Dienstleister Zugriff auf die Software oder verarbeitete Daten?" las `body/Andere_Softwareloesungen_Zugriff` statt `body/DL_Zugriff_auf_Software_Daten`. Korrektes Feld aus den Formularbindungen der App ermittelt (`ITSecurityFreigabenScreen.pa.yaml` Z. 2306/2474) — **kein SharePoint-Zugriff nötig, offene Kundenrückfrage damit erledigt**.

## Teil 0 – Flows aktivieren & testen (Maker-Portal, ~10 Min.)

> ⛔ **Blocker (Stand 07.08.):** Aktivieren scheitert mit `ConnectionAuthorizationFailed` auf `hse_EFABOSharePoint` — die SA-Connections (Besitzer `desvc.efabo@hse.com`) sind nicht mit `jan.mehlin_external` geteilt. Entsperren, eine der Optionen:
> - **A (sauber, sobald AVD HELP-78885 da):** Auf AVD als `desvc.efabo@hse.com` anmelden → make.powerapps.com → Connections → alle 3 EFABO-Connections (SharePoint, Office 365 Outlook, OneDrive) → Teilen → `jan.mehlin_external` mit „Kann verwenden" hinzufügen. Danach kann Jan aktivieren. Alternativ Flows direkt als SA einschalten.
> - **B (❌ getestet 07.08., scheitert):** Admin-PowerShell (`Set-AdminPowerAppConnectionRoleAssignment`, RoleName `CanView` — nicht `CanUse`!) liefert `Forbidden`: Jans Rechte erlauben Admin-Lesen, aber kein Connection-Sharing (Tenant-Rolle Power Platform Admin nötig). Script bleibt unten für einen HSE-Admin dokumentiert.
> - **C:** HSE-Admin (Power Platform Admin) teilt die 3 SA-Connections mit Jan — per PPAC/PowerShell-Block unten oder direkt als Besitzer. Bitte dazu in Mail an Maubach 07.08. Alternativ: Person mit bestehendem Connection-Zugriff schaltet beide Flows ein.
> - ❌ **Nicht:** Connection Reference `hse_EFABOSharePoint` auf eigene Connection umbiegen — wird von allen Flows der Solution geteilt, Berechtigungs-Flows brauchen SA-Rechte (Manage Permissions) auf der Site.
>
> ```powershell
> # Voraussetzung: Power Platform Admin / Dynamics 365 Admin Rolle (Jan hat sie NICHT — Forbidden)
> Install-Module Microsoft.PowerApps.Administration.PowerShell -Scope CurrentUser
> Add-PowerAppsAccount   # Login als Admin
> $env = "066791b7-04a5-43a6-b01d-b1d6b971e869"
> $jan = "656999fd-fa20-459d-9b08-a7c01c8531b2"  # Objekt-ID aus der Fehlermeldung
> # SA-Connections finden (Ersteller desvc.efabo):
> Get-AdminPowerAppConnection -EnvironmentName $env |
>   Where-Object { $_.CreatedBy.userPrincipalName -eq "desvc.efabo@hse.com" } |
>   Select-Object ConnectionName, ConnectorName, DisplayName
> # Für jede der 3 Connections (shared_sharepointonline, shared_office365, shared_onedriveforbusiness):
> Set-AdminPowerAppConnectionRoleAssignment -EnvironmentName $env `
>   -ConnectionName "<ConnectionName aus Liste>" -ConnectorName "<ConnectorName aus Liste>" `
>   -RoleName CanView -PrincipalType User -PrincipalObjectId $jan
> # Gültige RoleNames: CanView (= "Kann verwenden"), CanViewWithShare, CanEdit
> ```

> ⚠️ **Vor Aktivierung: v1.0.2.1 importieren — aber nur das Flows-Paket!** Die Flow-Sources enthalten seit 07.08. das Auslöser-Kopie-Feature (manueller Button → Klicker bekommt Mail in CC, Entscheidung Jan 07.08.) und seit 12.08. den F15-Feldfix (siehe unten). Der DEV-Stand v1.0.2.0 hat beides nicht.
>
> 🔴 **Nicht das volle Paket importieren!** `src/CanvasApps/` ist noch der Alt-Stand aus v1.0.1.5 (mit `btn_PrintAF`/`btn_PrintITSec` und `CreateHTML_PDF_*` als Datenquellen). Der App-Umbau vom 11.08. existiert **nur** in der Live-App/`coauthoring/` und ist **nicht** nach `src/` zurückgeschrieben. Ein Import mit CanvasApp würde Teil 1 + Teil 2 komplett zurückrollen und zusätzlich das Coauthoring-Flag löschen (P7).
>
> ✅ **Fertig gepackt: `export/EFABO_1.0.2.1_flowsonly.zip`** (98 KB, unmanaged, v1.0.2.1, 13 Flows + Env-Vars, CanvasApp-RootComponent `type="300"` entfernt). Unmanaged-Import = Upsert → die App in DEV bleibt unangetastet, Coauthoring-Flag bleibt gesetzt. Import:
>
> ```powershell
> pac auth create --deviceCode --environment 066791b7-04a5-43a6-b01d-b1d6b971e869 --name HSE24-Dev   # P8: Token hält nur 11 h
> pac solution import --path "export\EFABO_1.0.2.1_flowsonly.zip" --activate-plugins --publish-changes
> ```
>
> ⚠️ **Nach dem Import Flow-States prüfen:** Im Paket stehen `Neuer Antrag`, `Geänderter Vertrag`, `Geänderter Vertrag - DEV TEST` und beide neuen Druck-Flows auf `StateCode 0` (F10). Ein Import kann aktive Flows in DEV **abschalten**. Insbesondere `Neuer Antrag` danach kontrollieren — steht er aus, bekommen neu erstellte Test-EFABOs keine Item-Berechtigungen und keine Genehmiger-Mail.
>
> 📌 **Ab jetzt Flows und App getrennt deployen.** App-Änderungen gehen über Studio/Coauthoring, Flow-Änderungen über das Flows-only-Paket. Für das Prod-GoLive muss die App vorher aus DEV frisch exportiert werden (nach Speichern + Veröffentlichen), sonst reist der Alt-Stand mit.

1. [make.powerapps.com](https://make.powerapps.com/environments/066791b7-04a5-43a6-b01d-b1d6b971e869/solutions) → Solution **EFABO** öffnen.
2. **Connection References prüfen** (Zahnrad an jedem Flow bzw. Solution → Connection References): alle 3 müssen eine gültige Connection haben — `EFABO SharePoint`, `Office 365 Outlook EFABO`, **`OneDrive for Business EFABO`** (wichtig: unter wessen Account die OneDrive-Connection läuft — dort landen die temporären HTML-Dateien und von dort läuft die PDF-Konvertierung. Ziel: `desvc.efabo@hse.com`).
3. Flow **„EFABO Druck und Versand"** öffnen → **Einschalten**.
4. Flow **„[Parent] EFABO Druck und Versand"** öffnen → **Einschalten**.
5. **Testlauf:** „EFABO Druck und Versand" → Testen → Manuell → `EfaboID` = ID eines abgeschlossenen Test-EFABOs, `Anlass` = `Manuell`. Erwartung: Mail an `legalcoordinator@hse.com` mit 1–2 PDFs (AF/ITSec, je nachdem was hinterlegt ist). ⚠️ `hse_EFABOTesting` steht in DEV auf `true` → Betreff hat „TEST: "-Präfix.
6. PDF-Layout prüfen (volle Breite, 40 %-Fragenspalte, keine Überlappungen) → Beispiel-PDF sichern und **per Mail an Annette Brunner zur Freigabe schicken** (Feindesign-Abstimmung, Entscheidung Jan 07.08.).
7. **„Geänderter Vertrag" bleibt vorerst AUS** (war in DEV schon deaktiviert). Auto-Druck-Test später: Flow einschalten, EFABO auf „Abgeschlossen" setzen. **„Geänderter Vertrag - DEV TEST" niemals gleichzeitig einschalten (Doppel-Feuer).**

## Teil 1 – App-Umbau (Pflicht, ~20 Min.)

> ⛔ **Blocker (Stand 07.08., P6 in PROBLEME.md):** Jan hat keinen Zugriff auf die SharePoint-Site `EFABO_DEV` → Studio lädt die Datenquellen nicht, Bearbeitung und Test blockiert. SA-Connections sind fürs Studio **nicht** nötig (Consent-Dialog „Zulassen" erstellt eigene Connections) — es fehlt nur der Site-Zugriff. Entsperren: Site-Owner nimmt `jan.mehlin_external@hse.com` als Mitglied (Bearbeiten) auf + in die EFABO-SharePoint-Gruppen (Recht/IT-Security) für Rollentests. Selbstservice: Site-URL öffnen → „Zugriff anfordern". **Update 11.08.: Berechtigungen beantragt, läuft.**
>
> ⚠️ **Coauthoring nach jedem Import neu aktivieren (P7):** Jeder Solution-Import, der die App enthält (v1.0.2.0 ✔ passiert, künftig v1.0.2.1, Prod), setzt das Coauthoring-Flag zurück. Vor MCP-Arbeit: Studio → App öffnen → Einstellungen → **Co-Authoring** aktivieren → speichern. Live-App-GUID DEV für `connect`: `82f0abdb-6e73-46b5-a033-2611cea27ef9`.

App **„Erfassungsbogen Verträge"** im Studio öffnen (Bearbeiten).

### 1.1 Neuen Flow als Datenquelle hinzufügen

Linke Leiste → Power Automate (Blitz-Symbol) → **„Flow hinzufügen"** → **`[Parent] EFABO Druck und Versand`** auswählen. (Erscheint nur, wenn Teil 0 Schritt 4 erledigt ist.)

### 1.2 Neuer Druck-Button auf ViewEFABOScreen

**ViewEFABOScreen** → Einfügen → Schaltfläche, Name: **`btn_PrintEFABO`**. Platzierung: oberer Bereich neben dem Titel (frei nach Layout).

**OnSelect:**
```
UpdateContext({cvarPrintResult: '[Parent]EFABODruckundVersand'.Run(EfaboAntrag.ID, User().Email)});
If(
    IsBlank(cvarPrintResult.message),
    Notify(
        "Der Export ist fehlgeschlagen. Bitte versuchen Sie es erneut oder wenden Sie sich an den Support.",
        NotificationType.Error,
        8000
    ),
    Notify(cvarPrintResult.message, NotificationType.Success, 6000)
)
```

**Visible:**
```
(varRolleUser = "Prüfende Rechtsanwälte" || varRolleUser = "Sachbearbeiter Recht") &&
(EfaboAntrag.Status.Value = "Abgeschlossen" || EfaboAntrag.Status.Value = "Abgelehnt")
```

**Text:** `"Prüfdokumente an Legal Coordinator senden"`
**DisplayMode:** `=DisplayMode.Edit`

### 1.3 Alte Druck-Controls entfernen

| Screen | Control | Was |
|---|---|---|
| ArbeitsrechtlichteFreigabenScreen | `btn_PrintAF` | löschen |
| ArbeitsrechtlichteFreigabenScreen | `Icon5` (DocumentPDF) | löschen |
| ArbeitsrechtlichteFreigabenScreen | `Icon5_1` (Clock, bei X≈278) | löschen |
| ITSecurityFreigabenScreen | `btn_PrintITSec` | löschen |
| ITSecurityFreigabenScreen | `Icon5_2` | löschen |
| ITSecurityFreigabenScreen | `Icon5_3` (Clock) | löschen |
| ITSecurityFreigabenScreen | `btn_APVerlängern_1` | löschen (wirkungsloser toter Button = Bug B4) |

Suche im Strukturbaum (Ctrl+F im Baum) nach den Control-Namen. Nach dem Löschen: App-Prüfer (Stethoskop) öffnen → es dürfen **keine** Formelfehler auf gelöschte Controls verweisen.

### 1.4 Speichern & Veröffentlichen

Datei → Speichern → Veröffentlichen. Danach Smoke-Test als Jurist: Button sichtbar auf abgeschlossenem EFABO? Klick → Erfolgs-Notify + Mail?

### 1.5 Nach erfolgreichem Test (GoLive-Vorbereitung)

- Alte Flows **CreateHTML_PDF_AF** und **CreateHTML_PDF_ITSec deaktivieren** (nicht löschen — Rollback-Reserve bis GoLive).
- „Geänderter Vertrag" aktivieren für Auto-Druck-Test.

## Teil 2 – Bugfixes in der App (empfohlen, ~15 Min.)

### 2.1 Rollenermittlung case-sensitiv (Bug B4-Mitursache)

**App → OnStart**, Abschnitt `Concurrent(...)` mit den Rollen-Ifs (ca. Zeile 233/240):

Alt:
```
Lower(EfaboAntrag.'Verantwortlicher Mitarbeiter'.Email) = User().Email,
```
Neu:
```
Lower(EfaboAntrag.'Verantwortlicher Mitarbeiter'.Email) = Lower(User().Email),
```

Alt:
```
User().Email = Lower(EfaboAntrag.'Genehmiger (gem. Orga-Unterschriften)'.Email) Or User().Email = "desvc.efabo@hse.com",
```
Neu:
```
Lower(User().Email) = Lower(EfaboAntrag.'Genehmiger (gem. Orga-Unterschriften)'.Email) Or Lower(User().Email) = "desvc.efabo@hse.com",
```

### 2.2 ViewForm.OnFailure zeigt falsches Formular

**ViewEFABOScreen → ViewForm → OnFailure** — `EditForm.Error` (liegt auf anderem Screen!) durch `ViewForm.Error` ersetzen:
```
Notify(
    "Error: " & ViewForm.Error,
    NotificationType.Error,
    3000
);
```

### 2.3 AF_Form.OnFailure ist stumm

**ArbeitsrechtlichteFreigabenScreen → AF_Form → OnFailure** ergänzen:
```
Set(varStatus, Blank());
Notify(
    "Die Arbeitsrechtliche Freigabe konnte nicht gespeichert werden: " & AF_Form.Error,
    NotificationType.Error,
    8000
)
```

### 2.4 btn_EntwurfAbschicken setzt NextStatus nie

**EditEFABOScreen → btn_EntwurfAbschicken → OnSelect**, am Ende des Blocks stehen nackte Record-Literale. Alt:
```
{NextStatus: "In Prüfung"},
{NextStatus: "Prüfung Genehmiger"}
```
Neu (in `UpdateContext` einpacken):
```
UpdateContext({NextStatus: "In Prüfung"}),
UpdateContext({NextStatus: "Prüfung Genehmiger"})
```

### 2.5 Bug B3 (Anhang-Upload sporadisch) — Minimal-Fix, fest eingeplant (Entscheidung Jan 11.08.)

Ursache (F9-Verdacht, nicht reproduziert): `SubmitForm(...)` und direkt folgendes `Patch(...Attachment.Updates)` schreiben quasi gleichzeitig auf dasselbe Item (Race/ETag-Konflikt); Fehler verpuffen stumm, weil kein `IfError` um die Patches liegt.

**Minimal-Fix (~1 h, separate Session, erst NACH Smoke-Test von Teil 0/1 — nicht zwei Änderungen gleichzeitig ins Feld):**

1. Attachment-`Patch` aus dem Button-`OnSelect` in das `OnSuccess` des jeweiligen Formulars verschieben (läuft dann garantiert erst nach erfolgreichem Submit).
2. Jeden verschobenen Patch einpacken in:
   ```
   IfError(
       Patch(...),
       Notify("Anhang-Upload fehlgeschlagen: " & FirstError.Message, NotificationType.Error, 8000)
   )
   ```

Betroffene Buttons (4): `btn_ÄnderungenSpeichern`, `btn_EntwurfAbschicken` (beide EditEFABOScreen), Absenden-Button AF-Screen, Absenden-Button ITSec-Screen.

**Bewusst NICHT im Minimal-Fix** (nur dokumentiert, Details `Analyse/02-canvas-app.md` §4; voller Umbau 2–3 h, nur falls B3 nach Minimal-Fix erneut auftritt):
- Hilfsformulare `EditForm_Attachment`/`ViewForm_Attachments` ohne `Item`-Property (Updates ggf. leer)
- restliche ~6 Patch-Aufrufstellen ohne `IfError` (u. a. View-Misch-Patches)
- gleiche Race in `btn_VerlängerungBeantragen` (Status-Patch nach SubmitForm)
- `MaxAttachmentSize: 50` MB vs. tatsächliche SharePoint-Limits

## Rollback

- App: Details → Versionen → vorherige Version wiederherstellen.
- Flows: alte Druck-Flows sind unangetastet; „EFABO Druck und Versand" einfach deaktivieren; Solution-Baseline v1.0.1.5 liegt als Zip im Git (`export/` lokal) und Repo-Stand `8c41546`.
