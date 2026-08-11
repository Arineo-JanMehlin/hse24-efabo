# HSE24 EFABO – Umsetzungsleitfaden

> Konsolidiert aus: `Anforderungen/Neue Anforderungen 05.08.26.md`, `Anforderungen/Termin am 06.08.26.md`, E-Mail „WG: Bitte um Aufwandschätzung und Angebot" (A. Brunner, HSE, 20.01.2026 / weitergeleitet 06.08.2026).
> Stand: 07.08.2026

---

## 1. Kontext

- **Lösung:** EFABO (Dataverse-Solution, DEV-Env `066791b7-04a5-43a6-b01d-b1d6b971e869`)
- **Auth-Account:** `jan.mehlin_external@hse.com`
- **Service Account (Connections):** `desvc.efabo@hse.com`
- **Repo:** https://github.com/Arineo-JanMehlin/hse24-efabo.git
- **Stakeholder Kunde:** Annette Brunner (Manager Data Protection/Legal), Günther Sailer

## 2. Beauftragter Scope (Angebot)

| # | Position | Aufwand (CS = h) | Status |
|---|----------|------------------|--------|
| 1 | Anpassung Drucklayout | 2,00 | offen |
| 2 | Drucklogik für EFABO Hauptformular | 8,00 | offen |
| 3 | Neue Workflow-Logik: automatischer Druck + Benachrichtigung bei EFABO-Status „Abgeschlossen" oder „Abgelehnt" | 3,00 | offen |
| 4 | Entfernen Druckbuttons in App | 0,50 | entfällt ggf. (geht in #2/#3 auf, Fallback-Button bleibt) |
| 5 | Test und GoLive | 1,50 | offen |
| 6 | Projektmanagement / Abstimmung | 1,00 | laufend |
| | **Summe** | **16,00** | |

### Stundenbuchungen (intern, Arineo)

| Datum | Stunden | Buchungstext | Deckt ab |
|---|---|---|---|
| 07.08.2026 | 5,00 | „Neubau der Druck- und Versand-Flows: konsolidierter Flow für AF- und IT-Security-PDFs mit neuem Drucklayout, automatischem Versand an Legal Coordinator bei Abschluss/Ablehnung und Fehlerbehandlung" | Flow-Seite von #1–#3 komplett (inkl. Analyse, B2-Bugfixes F5/F6, Auslöser-Kopie-Feature) |
| 11.08.2026 | 3,00 | „App-Umbau via Coauthoring: neuer zentraler Druck-Button auf Detailansicht (Rollen-/Status-Sichtbarkeit, Flow-Anbindung vorbereitet), 7 veraltete Druck-/Verlängerungs-Buttons entfernt, 4 Review-Bugfixes übernommen, Auth-/Verbindungsprobleme gelöst und dokumentiert, App gespeichert und veröffentlicht" | App-Seite von #2 (Teil 1 + Teil 2 der Anleitung, außer 1.1 Flow-Anbindung/P4); Doku P6–P8 |
| | **Gebucht gesamt: 8,00** | **Restbudget: 8,00 von 16,00** | Restaufwand geschätzt ~5 h (Flow-Aktivierung+Test nach P4, Feindesign, B3-Fix, GoLive, PM) |

## 3. Anforderungen im Detail

### 3.1 Anpassung Drucklayout (#1)

Kunde unzufrieden mit PDF-Ausdruck. Konkrete Vorgaben (Termin 06.08. + E-Mail):

- [ ] Ganze Seitenbreite ausnutzen
- [ ] Linke Spalte vergrößern (2-spaltiges Layout)
- [ ] Texte der Spalten dürfen sich nicht überschreiben/überlappen
- [ ] Sinnvolle Umbrüche
- [ ] Insgesamt „ordentliches" Erscheinungsbild
- [ ] Abstimmung Feindesign mit Kunde (Demo-Termin)

**Technischer Ansatz:** HTML-Zusammensetzung im Flow überarbeiten („Zusammensetzung HTML Bau" aus offenen Aufgaben) – vermutlich HTML→PDF-Konvertierung im Druck-Flow.

### 3.2 Drucklogik EFABO Hauptformular (#2)

Ist-Stand: Druck-Buttons auf Unterformularen (Arbeitsrechtliche Prüfung, IT Security Prüfung), Mail geht an den Button-Drücker. Auf Prod instabil (siehe Bugs).

Ziel:

- [ ] Druck-Buttons aus Unterformularen entfernen
- [ ] Druckfunktion für **EFABO gesamt** (inkl. Arbeitsrechtliche Prüfung + IT Security Prüfung) für EFABOs mit Status „Abgeschlossen"/„Abgelehnt"
- [ ] Berechtigt: alle Juristen und Sachbearbeiter Bereich Legal
- [ ] **Fallback-Button** auf Startseite/Hauptformular behalten (Kunde will bei Fehlern manuell neu starten können) – Sichtbarkeit nach Arbeitsrechtlicher Prüfungslogik
- [ ] Präferenz laut Notizen: beide alten Buttons weg, **ein neuer Button mit neuem Flow** (statt Reparatur der alten – Prod-Probleme mit alten Flow-Referenzen)

### 3.3 Neue Workflow-Logik: Auto-Druck + Benachrichtigung (#3)

- [ ] Trigger: EFABO-Status wechselt auf „Abgeschlossen" oder „Abgelehnt"
- [ ] Haupt-Workflow ruft Druck-Flows automatisch als Subflows auf (kein Button-Druck nötig)
- [ ] Flows zusammenführen, Trigger anpassen (Konsolidierung aller bisherigen Druck-Flows)
- [ ] Mail an **Legal Coordinator** (`legalcoordinator@hse.com`) statt an Button-Drücker
- [ ] Anhang: PDF(s) – Arbeitsrechtliche Freigabe + IT Security (nur wenn für den EFABO hinterlegt)
- [ ] Mail-Inhalt:
  - EFABO-ID
  - Kommentar Legal („Lecare AZ:")
  - Kurzer Text mit „abgeschlossen" bzw. „abgelehnt"
- [ ] Fehlerhandling im Flow (Pflicht – bisheriger Print-Fehler zeigt nur Notify-Zeile ohne Meldung)

**Zu klären (Annette, 06.08.):** Ob automatisierte Mail-Variante an definierte Adresse im Angebot bereits mitgeschätzt war → intern klären, neue Aufwände per Mail zur Freigabe an HSE senden.

### 3.4 Test & GoLive (#5)

- [ ] Test auf DEV mit Kunde (Feindesign-Abstimmung)
- [ ] Deployment als managed Solution auf Prod
- [ ] Vorher: unmanaged Layer auf Prod prüfen/entfernen (siehe PROBLEME.md) – sonst überdecken sie das Update

## 4. Gemeldete Bugs (im Zuge der Umsetzung mitfixen)

| Bug | Beschreibung | Status |
|-----|--------------|--------|
| B1 | Print-Button funktioniert sporadisch nicht; nur kleine Notify-Zeile ohne Fehlermeldung. Auf Prod für manche User ok, für andere nicht. Workaround war unmanaged Layer (Flow in App entfernt/neu hinzugefügt). | offen – wird durch neuen Button + neuen Flow (#2/#3) obsolet |
| B2 | Nachträgliches Setzen von „Einsatz Software" setzt Berechtigungen nicht korrekt; bei initialer Erstellung funktioniert es. | offen – Analyse nötig (vermutlich Sharing-/Berechtigungs-Flow triggert nur bei Create) |
| B3 | Anhang-Upload schlägt manchmal fehl. | offen – Analyse nötig |
| B4 | EFABO-Verlängerung (Button, szenarioabhängig eingeblendet) funktionierte zeitweise nicht, „ging dann wieder". | beobachten – Ursache unklar |

## 5. Ideen (NICHT Teil des Angebots – nur bei Restbudget/Zusatzfreigabe)

> Vor Umsetzung: Aufwand schätzen und von HSE freigeben lassen.

- **Reminder für Genehmiger:** Freigabe angefragt + nach 1 Woche nicht erfolgt → Reminder-Mail. Auch für Rollen-Freigaben (Finanzen usw.).
- **Mehr automatische Infomails.** Offen: An wen? Inhalt? Auslöser/Zeitpunkt?

## 6. Analyse-Aufträge an Solution-Export

- [ ] Alle Flows inventarisieren: Trigger, Zweck, Mail-Empfänger, Subflow-Beziehungen
- [ ] Druck-Flows identifizieren (Arbeitsrecht, IT Security) + HTML-Bau-Logik lokalisieren
- [ ] App: Druck-Buttons + Verlängerungs-Button + Sichtbarkeitslogik finden
- [ ] Berechtigungslogik „Einsatz Software" (B2) nachvollziehen
- [ ] Anhang-Upload-Mechanik (B3) prüfen
- [ ] Hinweise auf DEV/Prod-Drift suchen (User hat unmanaged-Layer-Fixes teils nachträglich in DEV nachgezogen – teils evtl. nicht)
- [ ] ⚠️ Unmanaged Layer selbst liegen auf **Prod** – im DEV-Export nicht sichtbar. Für vollständigen Layer-Check Prod-Zugriff nötig.

## 7. Vorgehen

1. ~~Anforderungen konsolidieren~~ ✅ (dieses Dokument)
2. ~~PAC Auth + Solution-Export aus DEV, entpacken~~ ✅ (`src/`, EFABO v1.0.1.5)
3. ~~Analyse~~ ✅ → `Analyse/01-flows.md`, `Analyse/02-canvas-app.md`, `Analyse/03-berechtigungen-solution.md`, Befunde F1–F20 in `PROBLEME.md`
4. **Diff + Arbeitspakete AP1–AP6:** `Analyse/04-umsetzungsplan.md`
5. Umsetzung je Position (#1–#3), Bugs B1–B4 mitfixen (Ursachen alle identifiziert)
6. Demo/Feindesign-Abstimmung mit Kunde
7. Test & GoLive (#5)
