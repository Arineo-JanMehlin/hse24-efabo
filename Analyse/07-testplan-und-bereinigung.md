# Testplan & Flow-Bereinigung (DEV-Abnahme + GoLive-Vorbereitung)

> Stand 21.08.2026. Deckt alle seit 07.08. getätigten Änderungen ab (Flows v1.0.2.0/2.1 +
> Web-API-Fixes vom 21.08., Canvas-App-Umbau Teil 1/2 + B3/F21/F22).
> Nach erfolgreichem Durchlauf: Bereinigung gemäß Abschnitt 6, danach Prod-Deployment
> gemäß [06-alm-drift-dev-prod.md](06-alm-drift-dev-prod.md).

## 0. Vorbereitung

| # | Schritt | Details |
|---|---|---|
| V1 | Flows aktivieren | `Neuer Antrag`, `Geänderter Vertrag`, `EFABO Druck und Versand` (Child), `[Parent] EFABO Druck und Versand` stehen in DEV auf **Entwurf** → alle vier aktivieren (als SA `desvc.efabo@hse.com`) |
| V2 | Env-Var prüfen | `hse_EFABOTesting` = `true` → Mail-Betreffe bekommen Präfix „TEST: “ |
| V3 | ⚠️ Mail-Empfänger | Der neue Druck-Flow sendet **hart an `legalcoordinator@hse.com`** — auch aus DEV. Vor dem Test HSE (Annette Brunner) vorwarnen oder Postfach-Regel abstimmen; Betreff-Präfix „TEST: “ hilft beim Filtern |
| V4 | Testdaten | 3 Anträge in *EFABO Anträge* (EFABO_DEV-Site): **(a)** mit AF-Item **und** ITSec-Item, **(b)** nur AF-Item, **(c)** ohne beides |
| V5 | Testuser | 1 User mit Rolle „Prüfende Rechtsanwälte“ oder „Sachbearbeiter Recht“, 1 User ohne diese Rollen; zum F8-Test Gruppenrolle in abweichender Schreibweise verfügbar |
| V6 | App-Version | Aktuelle App (Version 18.08.2026) im DEV-Player öffnen |

## 1. Druck-Feature (neu: Positionen #1–#3)

| ID | Prüfung | Schritte | Erwartet |
|---|---|---|---|
| T1 | Fallback-Button Sichtbarkeit | ViewEFABOScreen mit beiden Testusern öffnen, Anträge in Status Abgeschlossen/Abgelehnt und in anderem Status | `btn_PrintEFABO` nur sichtbar für Rollen „Prüfende Rechtsanwälte“/„Sachbearbeiter Recht“ **und** Status Abgeschlossen/Abgelehnt |
| T2 | Manueller Druck, Vollfall | Als berechtigter User bei Antrag (a) auf Drucken klicken | Erfolgs-Notify in App („…an den Legal Coordinator versendet. Eine Kopie ging an …“); Mail an `legalcoordinator@hse.com`, **CC an Klicker**; Betreff `TEST: EFABO <ID>: <status> – Prüfdokumente`; **2 PDF-Anhänge** (AF + ITSec) |
| T3 | PDF-Layout (#1) | Beide PDFs aus T2 öffnen | volle Seitenbreite, linke Spalte ~40 %, keine Text-Überlappung, saubere Zeilenumbrüche bei langen Antworten, kein Zeilen-Zerreißen am Seitenwechsel; korrekte Dateinamen (AF-PDF heißt „Arbeitsrechtliche…“, nicht „IT Security…“) |
| T4 | Mail-Inhalt (#3) | Mail aus T2 lesen | EFABO-ID, **Kommentar Legal („Lecare AZ“)** = Feld *Kommentar Recht*, Text „abgeschlossen“/„abgelehnt“ |
| T5 | Druck ohne ITSec (Ex-Bug B1) | Drucken bei Antrag (b) | genau 1 Anhang (AF-PDF), kein Fehler — alter AF-Flow scheiterte hier |
| T6 | Druck ohne Prüfdokumente | Drucken bei Antrag (c) | Mail ohne Anhang mit Hinweis „keine Prüfdokumente … hinterlegt“, kein Fehler |
| T7 | Fehlerpfad | Child-Flow kurz deaktivieren, Drucken klicken; danach reaktivieren | App zeigt Fehler-Notify (IfError), kein stiller Fehlschlag; Catch-Scope beendet Flow als Fehler |
| T8 | Auto-Druck Abgeschlossen | Antrag (a): Status auf „Abgeschlossen“ setzen (NeuerStatus=true) | „Geänderter Vertrag“ läuft: übliche Status-Mails **plus** Druck-Mail an legalcoordinator (CC leer, da kein Klicker); NeuerStatus danach zurückgesetzt |
| T9 | Auto-Druck Abgelehnt | dito mit „Abgelehnt“ | wie T8 mit Ablehnungstexten |
| T10 | F15-Feldfixes | In AF-PDF Zeile „Ende der Leistungserbringung“ (gefüllt vs. leer), in ITSec-PDF Frage „Haben Dienstleister Zugriff…“ | Ende-Datum erscheint korrekt (prüfte früher fälschlich das Beginn-Feld); ITSec-Frage zeigt `DL_Zugriff_auf_Software_Daten`, nicht mehr den Wert der Frage darüber |

## 2. Berechtigungs-Fixes (B2, F6)

| ID | Prüfung | Schritte | Erwartet |
|---|---|---|---|
| T11 | ITSec-Zweig Child EzP (B2) | Antrag mit „Einsatz Software = ja“ aus der App von Entwurf → Prüfung geben | ITSec-Item wird behandelt (Status „In Prüfung“, Berechtigungen für ITSec-Gruppe gesetzt); Antrag ohne Software-Einsatz: Zweig wird übersprungen, Verhalten wie bisher |
| T12 | Berechtigungen ändern (F6) | In der App neuen Genehmiger auf einem Antrag setzen; danach einen User entfernen | neuer User erhält RoleDef **Pruefen** (1073741927 in DEV) auf dem Item; Remove entfernt Zuweisung; keine `getbyemail('Genehmiger')`-Fehler in der Run-Historie |

## 3. Canvas-App-Fixes

| ID | Prüfung | Schritte | Erwartet |
|---|---|---|---|
| T13 | Alte Controls weg | AF-/ITSec-Screens inspizieren | btn_PrintAF, btn_PrintITSec + 4 Druck-Icons und toter Verlängern-Button (btn_APVerlängern_1) nicht mehr vorhanden |
| T14 | F8/B4 Verlängerung (Lower-Fix) | Verlängerung mit User beantragen, dessen Rollen-/Gruppenname in abweichender Groß-/Kleinschreibung vorliegt | Rollenprüfung greift case-insensitiv, Verlängerungs-Funktion nutzbar |
| T15 | B3 Anhang-Upload | An allen 4 Absende-Buttons (Antrag absenden, AF/ITSec-Formulare) Datei anhängen und absenden | Anhänge hängen zuverlässig am richtigen Item (Patch läuft in `OnSuccess`); bei provoziertem Fehler erscheint Notify statt Silent Fail (IfError) |
| T16 | Fehler-Handler (2.2–2.4) | Absenden mit provoziertem Formularfehler (z. B. Pflichtfeld leer manipulieren oder offline) | Notify zeigt `ViewForm.Error` bzw. AF_Form-Fehlertext; Statusvariable `NextStatus` bleibt konsistent (kein falscher Folgestatus nach Fehler) |
| T17 | F21 ComboBox | Betroffene Personen-ComboBox öffnen | Einträge zeigen Namen (nicht leer — Picture-Feld ist aus DisplayFields raus) |
| T18 | F22 Hinweistext | Screen mit Vertragsvolumen-Feld öffnen | Hinweistext dauerhaft sichtbar, unabhängig vom Feldzustand |

## 4. Regression (CR-Umstellung f07a0 + Web-API-Fixes 21.08.)

Alle 11 Flows laufen jetzt über die SA-Verbindung `hse_sharedsharepointonline_f07a0`;
Reaktivierung hat die Bindung nur formal validiert — je ein Laufzeittest:

| ID | Flow | Schritte | Erwartet |
|---|---|---|---|
| T19 | Neuer Antrag | neuen Antrag in der App anlegen | Item-Berechtigungen gesetzt, Genehmiger-Mail (mit „TEST: “), Entwurf-Fall setzt nur Freigaben zurück; Run-Historie grün |
| T20 | Entwurf zu Prüfung (Parent+Child) | Antrag zur Prüfung geben | Berechtigungen (SB-Recht/Recht/Finance/Einkauf als Prüfer, Workermanagement lesend), Genehmiger-Mail |
| T21 | Check if user in SharePoint Group | App-Anmeldung mit beiden Testusern | Rollenerkennung korrekt (steuert u. a. T1) |
| T22 | EFABO Entwurf erstellen | „Als Entwurf kopieren“ in der App | neuer Entwurf mit kopierten Feldern, `draft_id` zurück |
| T23 | CreateHTML_PDF_AF (Rollback-Reserve) | einmal manuell im Maker-Portal ausführen (UserMail + gültige AF-ID) | läuft fehlerfrei nach Deklarations-Fix vom 21.08.; PDF-Name „Arbeitsrechtliche Fragen…“; danach Kandidat für Deaktivierung (→ 07) |
| T24 | Element archivieren | **Vorsicht: löscht Originale nach Archiv-Kopie.** Nur mit Wegwerf-Testitem `Created` älter 365 Tage manuell ausführen — oder Lauf ohne qualifizierende Items als Smoke-Test | Archiv-Kopie + Löschung des Testitems bzw. Leerlauf ohne Fehler |
| T25 | Geänderter Vertrag, übrige Cases | Kommentar-Fall (NeuerKommentar=true) und einen weiteren Status (z. B. „In Prüfung“) durchspielen | Kommentar-/Status-Mails wie bisher, Teams-Posts funktionieren, Reset der Flags |

Nicht laufzeitrelevant (nur Designer): Schema-Fix `Parse_Arbeitsrechtliche_Prüfung` in
„Neuer Antrag“ — T19 deckt den Flow ab; im Designer stichprobenhaft prüfen, dass
Dynamic Content der Aktion wieder Felder anbietet.

## 5. Abschluss

1. Ergebnisse je Testfall dokumentieren (OK / Defekt + Run-Link).
2. Bereinigung ausführen: Abschnitt 6.
3. Prod-Deployment vorbereiten: Prod-Defekte, Env-Vars (Gruppen-IDs, **RoleDefIds über
   Kreuz!**), unmanaged Layer, Upgrade-Import, CR-f07a0-Bindung —
   alles in [06-alm-drift-dev-prod.md](06-alm-drift-dev-prod.md).

## 6. Bereinigung nach erfolgreichem Test

> Entscheidung Jan (07.08.): **vorerst nichts löschen** — dieser Abschnitt ist die
> Wiedervorlage dafür, sobald der Test oben erfolgreich abgeschlossen ist.

### Löschkandidaten

| Flow | WorkflowId | Status DEV | In Prod? | Begründung | Aktion nach Test |
|---|---|---|---|---|---|
| `Test FLow Anlage` | `221ba113-d6b6-ef11-b8e8-6045bd916686` | **Aktiviert (!)** | ja (identisch) | Testartefakt: eine GetItems-Aktion mit hartkodiertem `Title eq 174`, Ergebnis ungenutzt, keine Referenzen | sofort deaktivieren, nach Test löschen |
| `Geänderter Vertrag - DEV TEST` | `e7b63977-2fd1-ef11-a72e-6045bd932588` | Entwurf | ja (identisch) | veraltete Testkopie von „Geänderter Vertrag“ (ohne ITSec-Case, Teams-Posts auf Testkanal), hinkt funktional hinterher | löschen |
| `CreateHTML_PDF_AF` | `c4ec8987-5ebe-ed11-83fe-6045bd8f651f` | Aktiviert | ja | ersetzt durch „EFABO Druck und Versand“; die umgebaute App ruft ihn nicht mehr auf (btn_PrintAF entfernt) | nach Test **deaktivieren** (Rollback-Reserve), **löschen erst nach GoLive** |
| `CreateHTML_PDF_ITSec` | `1249cf9f-faf2-ef11-be20-6045bd909f2f` | Aktiviert | ja | dito (btn_PrintITSec entfernt) | nach Test **deaktivieren** (Rollback-Reserve), **löschen erst nach GoLive** |

### Reihenfolge und Stolpersteine

1. **Erst deaktivieren, später löschen.** Die beiden `CreateHTML_PDF_*`-Flows sind die
   Rollback-Reserve, falls der neue Druck-Flow nach GoLive Probleme macht
   (Plan-Entscheidung in [04-umsetzungsplan.md](04-umsetzungsplan.md), Abschnitt B).
2. **Prod wird nur per Upgrade-Import sauber.** Ein Update-Import lässt in DEV gelöschte
   Komponenten in Prod stehen. Für die endgültige Entfernung in Prod: Flows in DEV aus der
   Solution löschen → Export → **Stage-and-Upgrade**-Import in Prod.
3. **Prod-App zuerst.** Solange in Prod noch die alte App-Version läuft, referenziert sie
   `CreateHTML_PDF_AF/ITSec` (Buttons btn_PrintAF/btn_PrintITSec). Die beiden Flows in Prod
   erst löschen/deaktivieren, wenn die neue App-Version deployed ist.
4. **Env-Var `hse_ITSecurityFreigabe` bleibt.** Sie wird weiterhin vom neuen Druck-Flow
   genutzt — nur die alten Flows fallen weg, nicht die Variable.
5. **Connection References bleiben alle.** `f07a0` (SharePoint), `44daf` (Office 365),
   `96b42` (OneDrive), `30bd2` (Teams) werden von verbleibenden Flows genutzt.
6. **Löschung als SA.** Alle Solution-Flows gehören inzwischen `desvc.efabo@hse.com`;
   Löschen per Maker-Portal (SA-Login) oder Web API.

### Nicht anfassen (außerhalb der EFABO-Solution)

Umgebungs-Flows, die nicht zu EFABO gehören: `SLAInstanceMonitoringWarningAndExpiryFlow`,
`Dynamics 365-Wissensartikelflow suchen`, `Trigger-Flow für integrierte Such-API`,
`Get System Jobs`, `Create SOX User Status Rows` (+ „Kopie von –“-Duplikat). Bleiben unberührt.

### Statuszielbild nach Bereinigung (DEV wie Prod)

| Flow | Soll-Status |
|---|---|
| Neuer Antrag, Geänderter Vertrag | **Aktiviert** (stehen in DEV aktuell auf Entwurf — für Test aktivieren!) |
| EFABO Druck und Versand (Child) + [Parent] EFABO Druck und Versand | **Aktiviert** |
| Element archivieren, EFABO Entwurf erstellen, Check if user in SharePoint Group, Parent/Child EzP, Parent/Child Berechtigungen ändern | Aktiviert (unverändert) |
| CreateHTML_PDF_AF, CreateHTML_PDF_ITSec | Entwurf (bis GoLive), dann gelöscht |
| Test FLow Anlage, Geänderter Vertrag - DEV TEST | gelöscht |

Offener Punkt aus [06-alm-drift-dev-prod.md](06-alm-drift-dev-prod.md): `Element archivieren`
steht in **Prod** auf Entwurf — klären, ob Absicht, sonst beim Deployment mit aktivieren.
