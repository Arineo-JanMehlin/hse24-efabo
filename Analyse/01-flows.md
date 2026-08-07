# Analyse: Power Automate Flows – EFABO

> Quelle: Solution-Export EFABO v1.0.1.5 (unmanaged, DEV), `src/Workflows/`. Stand 07.08.2026.
> Statuscodes aus `.data.xml`: `StateCode 1/StatusCode 2` = aktiviert, `StateCode 0/StatusCode 1` = Entwurf (deaktiviert).
> Alle Site-/Listen-Referenzen via Umgebungsvariablen; Defaults zeigen auf `https://homeshoppingeurope24.sharepoint.com/sites/EFABO_DEV`.
> Wichtige Listen: EFABO Anträge (`d8010e65-d7b0-4b11-b0a0-ac65f5b96208`), Arbeitsrechtliche Freigabe (`312ce00d-e31d-4435-8e4c-63e76f0e6dfa`), IT-Security Freigabe (`15e4a9aa-67e1-46a3-b61e-8d713d4c32f5`), Anträge-Archiv (`39e1c6f3-…`), AP-Archiv (`dd6503ba-…`).

## 1. CreateHTML_PDF_AF (Druck Arbeitsrechtliche Freigabe)

- WorkflowId `{c4ec8987-5ebe-ed11-83fe-6045bd8f651f}` · **aktiviert** (v1.0.0.9)
- **Trigger:** Request `PowerAppV2` – Inputs: `text` („UserMail"), `number` („ID"). Aufruf aus Canvas App (Button).
- **Zweck:** HTML-Tabelle aus Item der Liste *Arbeitsrechtliche Freigabe*, Konvertierung via OneDrive zu PDF, Mail an Aufrufer.
- **Connections:** OneDrive (`hse_sharedonedriveforbusiness_96b42`), Office 365 (`hse_sharedoffice365_44daf`), SharePoint (`hse_EFABOSharePoint`) – alle `runtimeSource: "invoker"` → laufen im Kontext des App-Nutzers (HTML/PDF landen in **dessen** OneDrive `/Downloads/`).
- **Ablauf:** Get_item AF → Get_item ITSec (nur für Title im Dateinamen!) → Compose HTML → Create file HTML (OneDrive) → Convert to PDF → Create file PDF → Send an email (V2) → Respond to PowerApp.
- **Mail:** To = `@triggerBody()['text']` (= Button-Drücker!), Subject = PDF-Dateiname, PDF als Anhang.
- **Fehlerhandling:** keins – alles `runAfter: Succeeded`; Respond nur bei Erfolg.
- **Bugs:**
  - PDF-Name/Betreff lautet „Export IT Security Fragen für EFABO (…)" mit Title aus ITSec-Item – Copy-Paste-Fehler; Inhalt ist AF. Nebeneffekt: **Flow scheitert, wenn kein ITSec-Item existiert** (Kandidat für sporadischen Print-Bug B1!).
  - Zeile „Ende der Leistungserbringung" prüft `empty(...Beginn_der_Leistungserbringung...)` statt `Ende_...`.

**HTML-Layout (Compose, wörtlich):**
```css
* { font-family: Arial; font-size: 11px; }
th,td { padding:6px; text-align: start; vertical-align: text-top; }
th { max-width: 100px; min-width: 100px; }
td { max-width: 340px; min-width: 340px; }
h3 { font-size: 20px; font-weight: bolder; padding-bottom: 4px; margin-bottom: 0px; }
td { background-color:#f3f3f3; }
```
Eine `<table>` mit `<tr><th>Frage</th><td>Antwort</td></tr>` (2-spaltig). Boolean via `if(equals(...,false),'Nein','Ja')`, Datum via `formatDateTime(...,'dd.MM.yyyy')`.
**Layout-Problemursachen:** feste min/max-width in px (100px/340px), kein `table-layout: fixed`, kein `width:100%`, kein `word-wrap/overflow-wrap`, keine `page-break`-Regeln, kein `border-collapse`.

## 2. CreateHTML_PDF_ITSec (Druck IT Security)

- WorkflowId `{1249cf9f-faf2-ef11-be20-6045bd909f2f}` · **aktiviert** (v1.0.1.4)
- Trigger/Connections/Ablauf/Mail/Fehlerhandling: identisch zu AF-Flow (Kopie), Quelle *IT-Security Freigabe*. To = Button-Drücker.
- **HTML:** gleiches CSS **plus zweiter, widersprüchlicher th-Block**: `th{ min-width: 150px; }` (überschreibt 100px-min-width, `max-width:100px` bleibt → **max < min** → genau die gemeldeten Überlappungs-/Umbruchprobleme). Abschnittsüberschriften als `<tr><th colspan="2">…</th></tr>`.
- **Bug:** Frage „Haben Dienstleister Zugriff…" nutzt fälschlich dasselbe Feld `Andere_Softwareloesungen_Zugriff` wie Frage darüber.

**Konsolidierungs-Basis:** Beide Flows ~90 % identisch. Gemeinsamer Flow mit Typ-Parameter trivial machbar. Andockstelle für Auto-Druck: „Geänderter Vertrag" Switch-Cases `Abgeschlossen`/`Abgelehnt`. **Achtung:** `runtimeSource: invoker` funktioniert nur bei App-Aufruf – bei automatischem Aufruf Connection-Referenzen auf festes Konto (Service Account) umstellen. Mail-Empfänger künftig `legalcoordinator@hse.com` statt `triggerBody()['text']`.

## 3. Neuer Antrag

- WorkflowId `{79b669e9-7d48-ec11-8c62-000d3a4424d4}` · **DEAKTIVIERT (Entwurf) im Export!**
- **Trigger:** SharePoint `When_an_item_is_created` auf *EFABO Anträge*, Polling 5 Min.
- **Zweck:** Initialisiert neue Anträge: Item-Level-Berechtigungen (UnshareItem + `addroleassignment` per SP-HTTP für Genehmiger + Gruppen Recht/SB-Recht/Finance/Einkauf/ITSec/Workermanagement, RoleDef-IDs aus Env-Vars 1073741926/1073741927), Link-Feld, Genehmiger-Mail; bei Status „Entwurf" nur Freigaben zurücksetzen + Terminate(Succeeded).
- **Mail:** To = `Genehmiger/Email`, Subject `…(ID:) - Genehmigen/Ablehnen` – **Bug: ID im Betreff leer** (Zeile 2793). `varTesting`-Präfix „TEST: " wenn `hse_EFABOTesting`=true.
- **Fehlerhandling:** keins.

## 4. Geänderter Vertrag (Prod-Variante)

- WorkflowId `{95854567-1a49-ec11-8c62-000d3a442ccc}` · **DEAKTIVIERT (Entwurf) im Export!**
- **Trigger:** SharePoint „Wenn ein Element geändert wird" auf *EFABO Anträge*, 5-Min-Polling, Trigger-Condition `@or(equals(triggerBody()?['NeuerStatus'],true),equals(triggerBody()?['NeuerKommentar'],true))`.
- **Zweck:** Zentrale Benachrichtigungsmaschine: Kommentar-Mails (Switch `KommentarVon`: Legal/Finance/Genehmiger/Einkauf/Antragsteller/LegalAP/ITSec/…) + Status-Mails/Teams-Posts (Switch `Status/Value`: **In Prüfung / Abgelehnt / Abgeschlossen / Zurückgezogen / Verlängerung beantragt / Verlängerung abgelehnt / Verlängert**); setzt am Ende `NeuerStatus`/`NeuerKommentar` zurück.
- **Connections:** SharePoint (2 Refs), Office 365 Outlook, **Teams** (`hse_sharedteams_30bd2`); Teams-Posts an Abteilungskanäle (Legal groupId `9aaebce9-…`, Finance `56c88fa1-…`).
- **Mails:** „Abgelehnt" → To = `VerantwortlicherMitarbeiter/Email`, Body mit `AblehnungsKommentar` (Switch `AbgelehntDurch/Value`: Genehmiger/Finance/Indirect Sourcing/Legal/ITSec/LegalAP); „Abgeschlossen" → To = `VerantwortlicherMitarbeiter/Email`, DE+EN-Body inkl. Legal-AP-/ITSec-Kommentar.
- **Fehlerhandling:** keins.
- **→ Hier (Cases `Abgeschlossen`/`Abgelehnt`) muss künftig der konsolidierte Druck-Flow angedockt werden.**

## 5. Geänderter Vertrag – DEV TEST

- WorkflowId `{e7b63977-2fd1-ef11-a72e-6045bd932588}` · **DEAKTIVIERT (Entwurf)**, v1.0.1.4.
- **Diff zu Prod-Variante** (programmatisch, metadata-bereinigt): Struktur ~90 % identisch. Unterschiede:
  1. SharePoint-Connection-Ref: `hse_EFABOSharePoint` → `hse_sharedsharepointonline_f07a0` (Dev-Verbindung)
  2. Alle Teams-Posts auf einen Test-Kanal umgebogen
  3. **Funktionaler Rückstand:** Ablehnungs-Switch **ohne Case `ITSec`**; „Abgeschlossen"-Mail mit älterem Body ohne IT-Security-Kommentarblock/Formatierung
  4. Kosmetik: neuere `$authentication`-Serialisierung
- **Fazit:** DEV/Prod-Drift bestätigt. DEVTEST = Testkopie, hinkt inhaltlich hinterher. **Löschkandidat**, keinesfalls weiterpflegen.

## 6./7. [Parent]+[Child] EFABO Entwurf zu Prüfung

- Parent `{1fce0777-…}` · aktiviert · Trigger PowerApp V1, Input EFABO-ID; ruft Child `b4019f00-…` mit `{number: EfaboID}`. Keine Response an App!
- Child `{b4019f00-…}` · aktiviert · Trigger Request/Button. Setzt beim Übergang Entwurf→Prüfung Item-Berechtigungen (Genehmiger + Gruppen SB-Recht 16 / Recht 15 / Finance 14 / Einkauf 12 als „Prüfer" 1073741927, Workermanagement 40 lesend 1073741826), Genehmiger-Mail oder Direkt-Patch; behandelt AF-Item (Status „In Prüfung", Berechtigungen, verwaiste AP-Items löschen).
- **Bug (kosmetisch):** Mail-Body doppelt HTML-encodiert (`&lt;p&gt;Hallo…`).
- Fehlerhandling: keins.

## 8./9. [Parent]+[Child] EFABO Berechtigungen ändern

- Parent `{93156a01-…}` · aktiviert · Trigger PowerApp V1; Inputs Role, NewUserMail, EfaboID, RemoveUserMail; ruft Child `da31cadf-…`.
- **⚠️ Mapping-Verdacht:** Parent mappt `text`=Role, `text_2`=NewUserMail; Child deklariert `text`=NewUserMail, `text_2`=Role → **Role/NewUserMail evtl. vertauscht** (funktioniert nur, wenn Belegung intern konsistent gedreht – prüfen!).
- Child `{da31cadf-…}` · aktiviert · `addroleassignment` für neuen User (Role='Genehmiger' → RoleDef „Pruefen" 1073741927, sonst „Erstellen" 1073741926); `removeroleassignment` wenn RemoveUserMail gesetzt.
- Fehlerhandling: keins, keine Fehler-Response.

## 10. Check if user in SharePoint Group

- `{d6fa9f3c-…}` · aktiviert · Trigger PowerApp V1 (UserPrincipalName, GroupName). SP-REST `_api/web/sitegroups/getByName('…')/Users?$filter=UserPrincipalName eq '…'` → Antwort `groupmember` (bool als String).
- **Einziger Flow mit Fehlerpfad:** Set-Variable false bei `runAfter: [Failed, Skipped]`; Response `runAfter: [Succeeded, Skipped]`.

## 11. EFABO Entwurf erstellen

- `{4fd67657-…}` · aktiviert · Trigger PowerApp V1 (GetEFABO_Id). Kopiert Antrag (GetItem → PostItem ~45 Felder, Status „Entwurf") als neuen Entwurf, gibt `draft_id` zurück. Fehlerhandling: keins.

## 12. Element archivieren

- `{58f29c0b-…}` · aktiviert · Trigger Recurrence wöchentlich So 12:00.
- Anträge `Created < -365d` → Archiv kopieren, Original löschen; AP-Item ggf. mit.
- **Risiken:** kein Fehlerhandling (Delete nach Create ohne Rollback/Alert); **archiviert nur AF, keine ITSec-Items** (Lücke seit ITSec-Erweiterung).

## 13. Test FLow Anlage

- `{221ba113-…}` · **aktiviert (!)** · Trigger Button ohne Inputs. Eine Aktion: GetItems auf ITSec-Liste mit hartkodiertem `Title eq 174`, Ergebnis ungenutzt.
- **Testartefakt / Löschkandidat.** Keine Referenzen darauf.

## Querschnittsbefunde

1. **Druck-PDF-Mail-Empfänger heute:** `To = @triggerBody()['text']` = Button-Drücker. Umstellung auf `legalcoordinator@hse.com` = 1 Literal (im konsolidierten Flow).
2. **Child-Flow-Landschaft:** nur 2 Referenzen (Parents → Childs). Druck-Flows sind **keine** Child-Flows (PowerAppV2-Trigger) → für Auto-Aufruf aus „Geänderter Vertrag" entweder Trigger umbauen oder Logik integrieren.
3. **Fehlerhandling:** außer „Check if user in SharePoint Group" hat **kein Flow** Failed-Pfade/Scopes/Fehler-Responses.
4. **Deployment-Risiko:** „Neuer Antrag", „Geänderter Vertrag" (+DEVTEST) stehen im Export auf **Entwurf/deaktiviert** – Import würde sie deaktiviert ausliefern.
5. **Weitere Bugs:** leere ID im Betreff (NeuerAntrag), falscher PDF-Name im AF-Flow, `max-width<min-width` im ITSec-CSS, doppelt kodiertes Mail-HTML (Child Entwurf→Prüfung), Role/NewUserMail-Mapping-Verdacht (Berechtigungen), doppelt verwendetes Feld im ITSec-HTML.
