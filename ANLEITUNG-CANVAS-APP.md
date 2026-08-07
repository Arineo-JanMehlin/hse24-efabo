# Anleitung: Canvas-App-Anpassungen EFABO (Studio)

> Stand 07.08.2026. Voraussetzung: Solution EFABO **v1.0.2.0** ist in DEV importiert (✅ erledigt).
> Reihenfolge einhalten — Teil 0 zwingend vor Teil 1!

## Teil 0 – Flows aktivieren & testen (Maker-Portal, ~10 Min.)

1. [make.powerapps.com](https://make.powerapps.com/environments/066791b7-04a5-43a6-b01d-b1d6b971e869/solutions) → Solution **EFABO** öffnen.
2. **Connection References prüfen** (Zahnrad an jedem Flow bzw. Solution → Connection References): alle 3 müssen eine gültige Connection haben — `EFABO SharePoint`, `Office 365 Outlook EFABO`, **`OneDrive for Business EFABO`** (wichtig: unter wessen Account die OneDrive-Connection läuft — dort landen die temporären HTML-Dateien und von dort läuft die PDF-Konvertierung. Ziel: `desvc.efabo@hse.com`).
3. Flow **„EFABO Druck und Versand"** öffnen → **Einschalten**.
4. Flow **„[Parent] EFABO Druck und Versand"** öffnen → **Einschalten**.
5. **Testlauf:** „EFABO Druck und Versand" → Testen → Manuell → `EfaboID` = ID eines abgeschlossenen Test-EFABOs, `Anlass` = `Manuell`. Erwartung: Mail an `legalcoordinator@hse.com` mit 1–2 PDFs (AF/ITSec, je nachdem was hinterlegt ist). ⚠️ `hse_EFABOTesting` steht in DEV auf `true` → Betreff hat „TEST: "-Präfix.
6. PDF-Layout prüfen (volle Breite, 40 %-Fragenspalte, keine Überlappungen) → Beispiel-PDF für Feindesign-Abstimmung mit HSE sichern.
7. **„Geänderter Vertrag" bleibt vorerst AUS** (war in DEV schon deaktiviert). Auto-Druck-Test später: Flow einschalten, EFABO auf „Abgeschlossen" setzen. **„Geänderter Vertrag - DEV TEST" niemals gleichzeitig einschalten (Doppel-Feuer).**

## Teil 1 – App-Umbau (Pflicht, ~20 Min.)

App **„Erfassungsbogen Verträge"** im Studio öffnen (Bearbeiten).

### 1.1 Neuen Flow als Datenquelle hinzufügen

Linke Leiste → Power Automate (Blitz-Symbol) → **„Flow hinzufügen"** → **`[Parent] EFABO Druck und Versand`** auswählen. (Erscheint nur, wenn Teil 0 Schritt 4 erledigt ist.)

### 1.2 Neuer Druck-Button auf ViewEFABOScreen

**ViewEFABOScreen** → Einfügen → Schaltfläche, Name: **`btn_PrintEFABO`**. Platzierung: oberer Bereich neben dem Titel (frei nach Layout).

**OnSelect:**
```
UpdateContext({cvarPrintResult: '[Parent]EFABODruckundVersand'.Run(EfaboAntrag.ID)});
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

### 2.5 Optional (Bug B3, Anhang-Upload sporadisch)

Ursache: `SubmitForm(...)` und direkt folgendes `Patch(...Attachment.Updates)` schreiben quasi gleichzeitig auf dasselbe Item (Race). Sauberer Fix = Patch in das jeweilige `Form.OnSuccess` verschieben. Betroffen: `btn_ÄnderungenSpeichern`, `btn_EntwurfAbschicken` (EditEFABOScreen), Absenden-Buttons AF-/ITSec-Screens. Aufwand ~1 h, empfehle separaten Termin — nicht im Schnelldurchlauf machen.

## Rollback

- App: Details → Versionen → vorherige Version wiederherstellen.
- Flows: alte Druck-Flows sind unangetastet; „EFABO Druck und Versand" einfach deaktivieren; Solution-Baseline v1.0.1.5 liegt als Zip im Git (`export/` lokal) und Repo-Stand `8c41546`.
