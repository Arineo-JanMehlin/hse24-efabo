# Analyse: Canvas App „Erfassungsbogen Verträge" (EFABO)

> Quelle: `src/CanvasApps/hse_erfassungsbogenvertrge_f6658_src/Src/*.fx.yaml`. Stand 07.08.2026. Zeilenangaben = Quelldatei-Zeilen.

## 1. Flow-Aufrufe (`.Run(`) — 31 Vorkommen, 6 Flows

| Flow | FlowNameId |
|---|---|
| CheckifuserinSharePointGroup | 731f47c7-2a01-4c71-8ba6-7085de535ff7 |
| CreateHTML_PDF_AF | d12f602c-fb46-4063-80bb-c748b89f8f44 |
| CreateHTML_PDF_ITSec | 00c4eac6-44e9-4cca-a397-1fcb5a6c8f21 |
| [Parent]EFABOEntwurfzuPrüfung | 5c9a178a-1066-46b9-9836-4b21a2844ba5 |
| [Parent]EFABOBerechtigungenändern | 84e09bf4-5b18-4c9a-ab69-b81b1d082137 |
| EFABOEntwurferstellen | a54a7827-b185-4402-8261-a54f0ae7428d |

Aufrufstellen:

| Screen | Control | Flow | Parameter |
|---|---|---|---|
| App.OnStart (Z. 247–295, Concurrent) | – | CheckifuserinSharePointGroup (5×) | `User().Email` + Gruppe („Sachbearbeiter Recht", „Prüfende Rechtsanwälte", „Sachbearbeiter Finanzen", „Einkauf", „IT Security") |
| ArbeitsrechtlichteFreigabenScreen | `btn_PrintAF` (Z. 4236) | CreateHTML_PDF_AF | UserMail + AF-Item-ID |
| ArbeitsrechtlichteFreigabenScreen | `btn_AbsendenNew` (Z. 2699/2820) | [Parent]EFABOEntwurfzuPrüfung | EfaboAntrag.ID |
| ITSecurityFreigabenScreen | `btn_PrintITSec` (Z. 90) | CreateHTML_PDF_ITSec | UserMail + ITSec-Item-ID |
| ITSecurityFreigabenScreen | `btn_AbsendenNew_ITS` (Z. 2877/3050) | [Parent]EFABOEntwurfzuPrüfung | EfaboAntrag.ID |
| EditEFABOScreen | `btn_ÄnderungenSpeichern` (Z. 5219, 4 Aufrufe) | [Parent]EFABOBerechtigungenändern (bis 2×) | Role, neueEmail, ID, alteEmail |
| EditEFABOScreen | `btn_TEST` (Z. 5415) | [Parent]EFABOBerechtigungenändern | **Visible false, toter Testbutton** |
| EditEFABOScreen | `btn_EntwurfAbschicken` (Z. 5481/5507) | [Parent]EFABOEntwurfzuPrüfung | EfaboAntrag.ID |
| EditEFABOScreen | `Button1_20`/`Button1_24` (Popup Kopie) | EFABOEntwurferstellen | → `.draft_id` |
| ViewEFABOScreen | `btn_GrGenehmigen` (Z. 5313) | [Parent]EFABOBerechtigungenändern | („Verantwortlicher", …) |
| ViewEFABOScreen | `btn_GrKommentarVersenden` (Z. 5394) | [Parent]EFABOBerechtigungenändern | Aufruf **nach** `Navigate(SuccessScreen)` – fragil |
| ViewEFABOScreen | `Button1_16`/`Button1_17` (Popup Kopie) | EFABOEntwurferstellen | → `.draft_id` |
| TestingScreen | `Button1_11`, `Button1_13` | CheckifuserinSharePointGroup | Testaufrufe |

Zusätzlich App.OnStart Z. 98: `'Office365-Gruppen'.ListGroupMembers("18d62c8a-…")` → `GenehmigerColl`.

## 2. Druck-Buttons

**`btn_PrintAF`** (AF-Screen Z. 4236–4272):
```
OnSelect:
  UpdateContext({ cvarMessage: CreateHTML_PDF_AF.Run(User().Email,
      LookUp('Arbeitsrechtliche Freigabe', Title = Text(EfaboAntrag.ID)).ID).message });
  Notify(cvarMessage, NotificationType.Success, 5000)
Visible:
  EfaboAntrag.Status.Value="Abgeschlossen" || "Zurückgezogen" || "Abgelehnt" || "Verlängerung beantragt"
```
- **Keine Fehlerbehandlung** (kein IfError/IsError); Notify immer `Success`. Flow-Fehler → `cvarMessage` leer → **leere Notify-Zeile** = gemeldeter Bug B1.
- **Visible-Formel defekt:** nur erster Vergleich gültig, Rest reine Strings im OR → Koerzierungsfehler; zuverlässig true nur bei „Abgeschlossen".
- Begleit-Icons `Icon5` (DocumentPDF) + `Icon5_1` (**Icon.Clock**, Überbleibsel) → `Select(btn_PrintAF)`.

**`btn_PrintITSec`** (ITSec-Screen Z. 90–123): identisches Muster, ohne IfError. `Visible: varRolleUser = "IT Security" || varRolleUser = "Sachbearbeiter Recht"`. Icons `Icon5_2`/`Icon5_3` (wieder Clock).

**Für neuen Fallback-Button:** „AF nötig"-Logik durchgängig `!dcv_GeneralFreigabe*.Value && !dcv_Whitelist*.Value && dcv_AufwVerg*.Value` (Suffix `_1`=ViewForm, `_2`=EditForm, ohne=NewForm), teils + `IsBlankOrError(LookUp('Arbeitsrechtliche Freigabe', Title=Text(EfaboAntrag.ID)))`.

## 3. Verlängerungs-Buttons (Bug B4)

- `btn_APVerlängern` (AF-Screen Z. 4289): `OnSelect: UpdateContext({cvarAPRenew: !cvarAPRenew})` – Toggle. `Visible: Status="Abgeschlossen" && varRolleUser="Antragssteller"`.
- **`btn_APVerlängern_1` (ITSec-Screen Z. 181): wirkungslos** – kein Control auf dem Screen konsumiert `cvarAPRenew`. → plausible B4-Ursache.
- `cvarAPRenew`-Reset in OnVisible beider Screens; true nur bei Status ∈ {Abgeschlossen, Verlängerung beantragt, Verlängerung abgelehnt, Verlängert} und `Ende_der_Leistungserbringung_Neu` nicht leer.
- `btn_VerlängerungBeantragen` (Z. 479): `SubmitForm(AF_Form)` + direkt `Patch('EFABO Anträge', …, {Status:"Verlängerung beantragt"})` → **Race** (SubmitForm asynchron). Notify-Text bei Datumsfehler irreführend.
- Genehmigung: `btn_ReGenehmigen_3` → „Verlängert"; `btn_ReAblehnenBestätigen_2` → „Verlängerung abgelehnt" + `'Abgelehnt durch': "LegalAP"`.
- **Weitere Sporadik-Kandidaten:** Rollenermittlung case-sensitiv (App.OnStart Z. 233/240: nur eine Seite `Lower()`) → `varRolleUser="Antragssteller"` scheitert bei Großbuchstaben in `User().Email`.

## 4. Anhang-Upload (Bug B3)

- NewEFABOScreen: Attachments-Card in NewForm (`MaxAttachments=15`, `MaxAttachmentSize=50`), Upload via SubmitForm.
- EditEFABOScreen: sichtbare Card `dcv_AnlagenEDIT` + **unsichtbares Hilfsformular `EditForm_Attachment`** (ohne `Item`!), Update = `dcv_AnlagenEDIT.Attachments`; Upload per `Patch('EFABO Anträge', LookUp(…), EditForm_Attachment.Updates)` (Aufrufe: AF Z. 2740/2818/3010, ITSec Z. 2916/3048/3216, Edit Z. 5259/5505).
- ViewEFABOScreen: analog `ViewForm_Attachments` (ohne `Item`), Misch-Patches Z. 5471–7552.

**Fehlerquellen:**
1. `SubmitForm(EditForm)` + `Patch(…Updates)` unmittelbar hintereinander auf dasselbe Item → **Schreibkonflikt/ETag (sporadisch)** = B3-Hauptverdacht
2. Kein IfError um Patches; Fehler verpuffen
3. Hilfsformulare ohne `Item` → Updates ggf. leer wenn Card-State nicht geladen
4. `MaxAttachmentSize: 50` MB – SharePoint-Limits können früher greifen

## 5. Rollen-/Sichtbarkeitslogik

- App.OnStart: `Set(varITSec, true)` Default; `GenehmigerColl` aus O365-Gruppe minus SP-Liste „Geschäftsführer"; **hartkodierte Testuser** (robert.mueller_external@hse.com, jan.mehlin_external@hse.com, desvc.efabo@hse.com, lisa.grueneisl@hse.com).
- Rollen-Flags per `CheckifuserinSharePointGroup.Run(…).groupmember = "True"` (Stringvergleich) in Concurrent; `colUserRollen` → `varRolleUser` + Navigation; bei mehreren Rollen StartScreen-Dropdown.
- AF-Edit-Berechtigung: `(varRolleUser="Prüfende Rechtsanwälte" || "Sachbearbeiter Recht") && (PruefungArbeitsrechtlicheFragen="In Prüfung" || Status="Verlängerung beantragt")`.
- **Schwächen:** einseitiges `Lower()`; Statusstring mit eingebettetem Zeilenumbruch `"⏎Prüfung Genehmiger"` (Z. 379–380) **matcht nie**; TestingScreen enthält abweichende Rollenlogik-Kopie.

## 6. Notify() ohne Fehlerdetails

1. `btn_PrintAF`/`btn_PrintITSec`: leerer Success-Notify bei Flow-Fehler (= B1)
2. `ViewForm.OnFailure` (Z. 83–88): referenziert **`EditForm.Error`** (anderer Screen!) → zeigt „Error: " ohne Inhalt
3. Kopie-Buttons: abgeschnittener Text „…kontaktieren Sie einen ", Tippfehler „Âdministrator"
4. `AF_Form.OnFailure`: **kein Notify** – Submit-Fehler komplett stumm
5. `NewForm.OnFailure`: detaillierter Error-Notify **auskommentiert**

## 7. Datenquellen

- SharePoint Site `…/sites/EFABO_DEV` (via Env-Vars): EFABO Anträge, Arbeitsrechtliche Freigabe, IT-Security Freigabe, Systemtexte, Geschäftsführer, EFABO Anträge (Archiv), EFABO Document Library
- Flows: die 6 o.g.; Office 365 Groups (nur OnStart)
- **Unbenutzt in App:** `'EFABO Anträge (Archiv)'`, `'EFABO Document Library'` – entfernbar als App-Datenquelle

## 8. Löschkandidaten / tote Controls

- **TestingScreen** (unerreichbar, kein Navigate), **Screen2** (leer), **AvailableColorsAndControlsScreen** (Styleguide, unerreichbar)
- `btn_TEST` (Visible false), `Label5`, `Label8` (Visible false), `btn_APVerlängern_1` (wirkungslos), `Icon5_1`/`Icon5_3` (Clock-Überbleibsel)
- **`btn_EntwurfAbschicken`** (Edit Z. 5508–5514): `If(…, {NextStatus:"In Prüfung"}, {NextStatus:"Prüfung Genehmiger"})` – **nackte Record-Literale ohne UpdateContext** → NextStatus wird nie gesetzt
- Delegation: keine gravierenden Fälle; `ListGroupMembers` liefert nur erste Seite (große Gruppen ggf. abgeschnitten)
