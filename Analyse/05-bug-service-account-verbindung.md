# Bug: EFABOs ab ID ~1602 hängen in „In Erstellung" / „Prüfung Genehmiger" – Formular für Genehmiger nicht öffenbar

> Gemeldet 19.08.2026 vom Kunden. Nur Recherche/Dokumentation – **keine Lösung**, da kein Zugriff auf die
> Service-Account-Connection (`desvc.efabo@hse.com`, siehe [[04-umsetzungsplan]]).

## Symptom

- Alle EFABO-Anträge ab ca. ID 1602 bleiben im SharePoint-Listenview auf Status **„In Erstellung"** (Spalte „Zum Formular")
  bzw. **„Prüfung Genehmiger"** (Spalte „Status") hängen – siehe Screenshot IDs 1603–1606.
- Der Genehmiger kann das Formular nicht öffnen → Freigabeprozess steht komplett.
- Betroffen: Flow **„Neuer Antrag"** (`NeuerAntrag-79B669E9-7D48-EC11-8C62-000D3A4424D4`), Trigger *„When an item is created"*
  auf der SharePoint-Liste „EFABO Anträge". Fehler tritt **jede** Ausführung ab dem genannten Zeitpunkt auf (letzte
  gesehene Ausführung: 19.08.2026, 11:27 Uhr, Dauer 20 ms → Abbruch quasi sofort).

## Root Cause (aus Fehlerdetails in Power Automate)

Fehlermeldung aus den Ausführungsdetails:

```
Error from token exchange: Runtime call was blocked because connection status is Enabled| Error,
and sharepointonline is in the block list. Connection errors:
[ParameterName: token, Error: Code: Unauthorized, Message: 'Failed to refresh access token for
service: sharepointonlinecertificatev2. Correlation Id=d62bd40d-6490-43d0-84fe-7f81fd9bab42,
UTC TimeStamp=8/19/2026 9:17:41 AM, Error: Failed to acquire token from AAD:
{"error":"invalid_grant","error_description":"AADSTS700003: Device object was not found in the
tenant '1d0f52d7-4151-4a31-...' directory. Trace ID: ad92-b176d150d4e6' Correlation ID:
6e5ed2c8-c996-4f9e-9fcb-ad99f697b493 Timestamp: 2026-08-19 09:17:41Z","error_codes":[700003],
...,"suberror":"device_authentication_failed", ...}]
```

Übersetzt:

1. **AADSTS700003 / `device_authentication_failed`**: Azure AD (Entra ID) findet für das Gerät, mit dem die
   `sharepointonlinecertificatev2`-Verbindung (Service Account **`desvc.efabo@hse.com`**) authentifiziert, kein
   Geräteobjekt mehr im Tenant `1d0f52d7-4151-4a31-…`. D.h. das für diese Connection hinterlegte Gerät/Zertifikat
   wurde im Entra ID **gelöscht, deaktiviert oder ist abgelaufen** (z. B. durch Stale-Device-Cleanup, Device-Compliance-Policy,
   Passwort-/Zertifikatsrotation des Service Accounts, oder eine Conditional-Access-Regel, die neuerdings ein
   compliant/hybrid-joined Gerät verlangt).
2. Als Folge kann Power Automate keinen frischen Access Token mehr für die `shared_sharepointonline`-Connection
   beziehen → die Connection wechselt in Status **Error** und wird deshalb von der Runtime **geblockt** ("connection ...
   is in the block list").
3. Der Trigger-Schritt selbst feuert noch (SharePoint-Item wird erkannt), aber **jeder nachfolgende SharePoint-Schritt**
   im Flow (`Set Link Feld`, `Freigabe eines Elements... beenden`, `Get UserPrincipalId`, `Set Verantwortlich`, …) scheitert
   sofort an derselben blockierten Connection → Flow bricht in den ersten Sekunden ab, das Item bleibt im
   Ausgangsstatus stehen, der Genehmiger-Link wird nie gesetzt/freigegeben.

**Wichtig:** Dies ist kein Logik-/Code-Fehler im Flow oder in der Canvas App, sondern ein **Infrastruktur-/Identitätsproblem
auf Ebene des Service-Accounts bzw. seiner Geräteregistrierung in Entra ID.** Der Flow selbst ist unverändert (kein
Deployment in diesem Zeitraum, siehe Commit-Historie); das Problem ist neu aufgetreten, seit die Connection Errors wirft.

## Warum "ab ID ~1602"

Der Fehler betrifft nicht die IDs selbst, sondern **den Zeitpunkt**, ab dem die Service-Account-Connection in den
Error-Status gefallen ist. Alle danach neu angelegten Items (ab ID ~1602/1603) laufen durch den fehlschlagenden Trigger-Flow
und bleiben hängen; ältere Items, die den Flow bereits vor dem Ausfall durchlaufen hatten, sind nicht betroffen.

## Betroffene Komponenten

- Flow: **Neuer Antrag** (`src/Workflows/NeuerAntrag-79B669E9-7D48-EC11-8C62-000D3A4424D4.json`)
- Connection: `shared_sharepointonline` (Service Account `desvc.efabo@hse.com`, `sharepointonlinecertificatev2`)
- Vermutlich **alle weiteren Flows**, die dieselbe Service-Account-SharePoint-Connection nutzen (z. B. Child/Parent
  EFABO-Flows, siehe [[01-flows]]), sobald sie nach dem Trigger-Item laufen – nicht nur „Neuer Antrag".

## Warum aktuell nicht lösbar (durch uns)

- Fix erfordert Zugriff auf:
  - die Connection selbst in Power Automate (Reconnect/Repair unter „Meine Verbindungen"), und/oder
  - das Geräte-/Kontoobjekt des Service Accounts `desvc.efabo@hse.com` in Entra ID (Device-Objekt neu registrieren,
    Konto-Login erneuern, Conditional-Access-Ausnahme prüfen).
- Beides liegt außerhalb unseres Zugriffs (kein Service-Account-Login, kein Entra-ID-Admin-Zugriff).

## Empfohlene nächste Schritte (für Kunde/IT-Admin)

1. In Power Automate (als Admin oder mit Zugriff auf die Service-Account-Connection) unter „Verbindungen" die
   `shared_sharepointonline`-Connection des Service Accounts `desvc.efabo@hse.com` prüfen → Status, ggf. **erneut anmelden /
   reparieren**.
2. In Entra ID (Azure AD) prüfen, ob für `desvc.efabo@hse.com` ein Geräteobjekt im Tenant `1d0f52d7-4151-4a31-…`
   existiert; falls gelöscht/abgelaufen: Gerät neu registrieren bzw. Konto neu anmelden, damit ein neues Geräteobjekt
   entsteht.
3. Prüfen, ob kürzlich eine Conditional-Access-Policy (Device Compliance / Hybrid Join) geändert wurde, die
   Service-Accounts ohne interaktives Gerät jetzt blockiert.
4. Nach Fix: betroffene Items ab ID ~1602 in der SharePoint-Liste manuell prüfen – ggf. müssen `Set Link Feld` /
   Freigabe / `Set Verantwortlich`-Schritte für bereits erstellte, aber nicht durchgelaufene Items **nachträglich
   manuell** ausgeführt oder die Items erneut angestoßen werden (Flow läuft nur beim `Item created`-Trigger, kein
   automatischer Retry).

## Status

**Ungelöst** – wartet auf Kunde/IT-Admin-Zugriff auf Service-Account-Connection bzw. Entra-ID-Geräteverwaltung.
