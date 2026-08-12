# Nachtrag Zusatzleistungen EFABO – Entwurf

> Stand 12.08.2026. Grundlage: Klärung P2 (`PROBLEME.md`), Ideen-Abschnitt 5 in `UMSETZUNGSLEITFADEN.md`, Befunde F13/F9 und AP4.3 in `Analyse/04-umsetzungsplan.md`.
>
> ⚠️ **Entwurf.** Aufwände sind Schätzungen von Jan, noch nicht intern geprüft (Robert Mueller) und nicht an HSE gesendet.

## 1. Klärung P2 – kein Nachtrag nötig

Annette Brunner hat im Termin 06.08. gefragt, ob die automatisierte Mail-Variante (Versand an eine **fest definierte** Adresse statt an den Button-Drücker) im Angebot über 16,00 CS bereits mitgeschätzt war.

**Ergebnis: ja, durch Angebotsposition #3 gedeckt.** Position #3 lautet wörtlich „Neue Workflow-Logik: automatischer Druck **+ Benachrichtigung** bei EFABO-Status ‚Abgeschlossen' oder ‚Abgelehnt'" mit 3,00 CS. Die Benachrichtigung ist damit Bestandteil der Beauftragung; der feste Empfänger `legalcoordinator@hse.com` ist im Flow ein einzelnes Literal, der Mehraufwand gegenüber „Mail an Auslöser" liegt bei praktisch null. Es geht also **kein** Nachtrag an HSE für diesen Punkt.

## 2. Tatsächlich außerhalb des Angebots

| # | Leistung | Schätzung (CS = h) | Grundlage |
|---|----------|--------------------|-----------|
| Z1 | **Reminder-Mails an Genehmiger.** Zeitgesteuerter Flow (tägliche Recurrence), der EFABOs mit offener Freigabe älter als 7 Tage ermittelt und eine Erinnerung an den jeweils zuständigen Genehmiger bzw. die Rollen-Freigeber (Finanzen, Einkauf, Recht, IT-Security) versendet. Inkl. neuem SharePoint-Feld für „letzte Erinnerung", damit nicht täglich erneut gemahnt wird, plus Fehlerhandling. | 4,00 | Ideen-Abschnitt 5 |
| Z2 | **Weitere automatische Infomails – Spezifikationstermin.** Empfänger, Inhalte und Auslösezeitpunkte sind derzeit offen. Vorschlag: 1 h Abstimmung mit HSE, danach belastbare Schätzung der Umsetzung. | 1,00 (+ Umsetzung offen) | Ideen-Abschnitt 5 |
| Z3 | **IT-Security-Items in die Archivierung aufnehmen (F13).** Der Flow „Element archivieren" archiviert bisher nur Arbeitsrechtliche-Freigabe-Items; IT-Security-Items bleiben seit der ITSec-Erweiterung liegen. Inkl. Fehlerhandling und Rollback für das Löschen nach dem Kopieren (aktuell wird gelöscht, ohne den Kopiervorgang zu prüfen). | 2,00 | F13 |
| Z4 | **„Einsatz Software" nachträglich setzen (B2-Restfall, AP4.3).** Wird das Feld erst nach Anlage aktiviert, laufen die Berechtigungen nicht nach. Umsetzung: beim Speichern im EditEFABOScreen den Wechsel false→true erkennen und den erweiterten Berechtigungs-Child-Flow aufrufen. Die im Angebot enthaltenen B2-Anteile (F5 ITSec-Block, F6 Mapping) sind bereits umgesetzt. | 2,50 | AP4.3 |
| | **Summe** | **9,50** | |

### Bedingter Zusatzpunkt (nicht anbieten, nur vormerken)

| # | Leistung | Schätzung |
|---|----------|-----------|
| Z5 | **Vollständige Stabilisierung Anhang-Upload (B3, F9).** Nur relevant, falls B3 nach dem im Restbudget enthaltenen Minimal-Fix erneut auftritt: Hilfsformulare `EditForm_Attachment`/`ViewForm_Attachments` mit `Item`-Property versehen, restliche ~6 Patch-Stellen mit `IfError` absichern, Race in `btn_VerlängerungBeantragen`, `MaxAttachmentSize` gegen die realen SharePoint-Limits abgleichen. | 2,00–3,00 |

## 3. Mailentwurf an HSE

> An: Annette Brunner · CC: Günther Sailer, Robert Mueller
> Betreff: EFABO – Rückfrage automatisierte Benachrichtigung, Statusupdate und optionale Zusatzleistungen

Hallo Frau Brunner,

kurzes Update zum EFABO-Projekt und die Antwort auf Ihre Rückfrage aus unserem Termin am 06.08.

**Zu Ihrer Rückfrage:** Die automatisierte Benachrichtigung an eine fest definierte Adresse (`legalcoordinator@hse.com`) statt an den jeweiligen Button-Drücker ist von unserem Angebot bereits abgedeckt – Position 3 umfasst ausdrücklich „automatischer Druck und Benachrichtigung bei EFABO-Status ‚Abgeschlossen' oder ‚Abgelehnt'". Es entstehen dafür also keine zusätzlichen Kosten, ein Nachtrag ist nicht erforderlich.

**Umsetzungsstand:** Die Flow-Seite ist fertig und in der Entwicklungsumgebung eingespielt. Es gibt jetzt einen konsolidierten Druck- und Versand-Flow, der das gesamte EFABO inklusive Arbeitsrechtlicher Prüfung und IT-Security-Prüfung ausgibt, mit dem überarbeiteten Drucklayout (volle Seitenbreite, größere linke Spalte, saubere Umbrüche) und Fehlerbehandlung. In der App ist der neue zentrale Druck-Button auf der Detailansicht angelegt, die alten Druck-Buttons auf den Unterformularen sind entfernt. Zusätzlich haben wir mehrere der gemeldeten Fehler mitbehoben.

**Wo wir Unterstützung von HSE brauchen**, um in den Test zu kommen:

1. Die drei Verbindungen des Service-Accounts `desvc.efabo@hse.com` (SharePoint, Outlook, OneDrive) müssen für meinen Zugang freigegeben werden – ohne das lassen sich die neuen Flows nicht einschalten.
2. Lesend/schreibender Zugriff auf die SharePoint-Site `EFABO_DEV` für Tests in der Anwendung (ist beantragt).

Sobald das vorliegt, erzeuge ich ein Beispiel-PDF und schicke es Ihnen zur Freigabe des Feindesigns, bevor wir auf die Produktivumgebung gehen.

**Optionale Zusatzleistungen:** Aus unseren Gesprächen sind einige Themen entstanden, die nicht Bestandteil des aktuellen Angebots sind. Falls Sie diese umsetzen möchten, sende ich gern ein separates Angebot:

- Erinnerungs-Mails an Genehmiger, wenn eine Freigabe länger als eine Woche offen ist (ca. 4 Stunden)
- Aufnahme der IT-Security-Prüfungen in die automatische Archivierung – diese werden derzeit nicht mitarchiviert (ca. 2 Stunden)
- Korrekte Berechtigungsvergabe, wenn „Einsatz Software" erst nachträglich aktiviert wird (ca. 2,5 Stunden)
- Weitere automatische Informationsmails: hier wäre zunächst ein kurzer Abstimmungstermin nötig, um Empfänger, Inhalte und Auslöser festzulegen (ca. 1 Stunde), danach kann ich den Umsetzungsaufwand benennen

Geben Sie mir gern Bescheid, welche Punkte für Sie interessant sind.

Viele Grüße
Jan Mehlin
Arineo GmbH
