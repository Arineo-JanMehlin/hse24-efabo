# Analyse: Berechtigungslogik, Connection References, Env-Variablen, Solution-Metadaten

> Quellen: `src/Workflows/`, `src/Other/Solution.xml` + `Customizations.xml`, `src/environmentvariabledefinitions/`, Canvas-App-Quellen (Verifikation). Stand 07.08.2026.

## 1. Berechtigungs-Architektur

Berechtigungen werden an **3 Stellen** gesetzt:
- (a) automatisch bei Item-Erstellung: Flow **„Neuer Antrag"** (Create-Trigger, keine Trigger-Conditions)
- (b) beim Absenden eines Entwurfs: **„[Child] EFABO Entwurf zu Prüfung"**
- (c) nachträgliche Genehmiger-/Verantwortlichen-Wechsel: **„[Parent]/[Child] EFABO Berechtigungen ändern"**

Der einzige Modified-Trigger-Flow („Geänderter Vertrag") setzt **keinerlei** Berechtigungen (0 Treffer `addroleassignment`).

### „Neuer Antrag" – Einsatz-Software-Behandlung

```json
"expression": { "equals": [ "@triggerOutputs()?['body/Einsatz_Software']", "@true" ] }
```
Wenn true: `PruefungITSec = "In Prüfung"` + Lookup ITSec-Freigabe-Item (`Title eq <EFABO-ID>`) + UnshareItem + addroleassignments (Verantwortlicher „Erstellen"; Genehmiger, SB Recht, Recht, Finance, Einkauf, ITSec-Gruppe 45 je „Prüfen").
Entwurfs-Pfad: bei Status „Entwurf" nur Freigabe-Reset → `Terminate(Succeeded)` – **komplette Prüfer-/ITSec-Kette übersprungen**.

### 🔴 Bug B2 – Ursache bestätigt (doppelte Lücke)

1. **Nachträgliches Setzen von `Einsatz_Software` triggert nichts Berechtigungsrelevantes.** ITSec-Berechtigungslogik existiert NUR im Create-Flow „Neuer Antrag" (wertet Creation-Snapshot aus). „Geänderter Vertrag" feuert bei Feld-Update meist gar nicht (Trigger-Condition nur `NeuerStatus`/`NeuerKommentar`) und hat ohnehin keine Berechtigungslogik.
2. **Entwurfspfad hat ITSec-Lücke:** „[Child] EFABO Entwurf zu Prüfung" repliziert die Berechtigungskette aus „Neuer Antrag" (inkl. Arbeitsrecht-Zweig), aber es **fehlen komplett**: `Set_ITSec_mit_Prüfer`, `Einsatz_Software`-Bedingung, `PruefungITSec`-Patch, Grants aufs ITSec-Freigabe-Item. Als Entwurf gestartete oder nachträglich um Einsatz-Software ergänzte Anträge bekommen nie ITSec-Berechtigungen → exakt Kundensymptom.

**Fix:** ITSec-Block analog „Neuer Antrag" in `ChildEFABOEntwurfzuPrfung-B4019F00….json` ergänzen; für Update-Fall klären, ob App beim Aktivieren von Einsatz-Software einen Instant-Flow aufrufen soll.

### 🔴 Zusatzbefund: Parameter-Mapping „[Parent] EFABO Berechtigungen ändern" gekreuzt

App ruft (WADL-verifiziert): `Run(Role, NewUserMail, EfaboID, RemoveUserMail)`. Parent mappt an Child:
```json
"text":   "@triggerBody()['Variableinitialisieren_Wert']",   // Role  → Child-NewUserMail  ❌
"text_1": "@triggerBody()['RemoveUserMail_Wert']",           // ✓
"text_2": "@triggerBody()['Variableinitialisieren2_Wert']",  // Email → Child-Role         ❌
"number": "@triggerBody()['EfaboID_Wert']"                   // ✓
```
Child ruft damit `getbyemail('Genehmiger')` auf (schlägt fehl); Rollen-Weiche vergleicht E-Mail mit „Genehmiger" (immer „Erstellen"). **Nachträgliche Genehmiger-/Verantwortlichen-Wechsel setzen aktuell keine korrekten Berechtigungen** – zweiter, unabhängiger Berechtigungs-Bug (gleiche Symptomklasse wie B2).

### Weitere Befunde

- „Check if user in SharePoint Group": `runtimeSource: "invoker"` → läuft als App-User; braucht Leserecht auf Site-Gruppen, sonst liefert Fehlerpfad immer `false`.
- Parents haben keine Connections (nur Child-Aufruf); Childs `runtimeSource: "embedded"` → Elevation über Connection-Reference-Bindung.
- „Neuer Antrag": Bedingung vergleicht `VerantwortlicherMitarbeiter/Email` mit Objekt `body/Author` (statt `Author/Email`) → immer ungleich, Grant läuft immer (harmlos).

## 2. Auto-Druck-Hook

Richtiger Hook: **„Geänderter Vertrag"**, Switch `Option_-_Abfrage_neuer_Status`, Cases **„Abgeschlossen"** / **„Abgelehnt"**. Achtung: Flow setzt am Ende `NeuerStatus`/`NeuerKommentar` zurück – neue Logik **vor** den Reset. Flow ist im Export deaktiviert (StateCode 0).

## 3. Status-/Felder-Inventar (SharePoint, interne Namen)

| Spalte | Typ | Werte |
|---|---|---|
| `Status` | Choice | Entwurf, Prüfung Genehmiger, In Prüfung, Abgeschlossen, Abgelehnt, Zurückgezogen, Verlängerung beantragt, Verlängerung abgelehnt, Verlängert |
| `PruefungRecht` | Choice | „Anlage Akte in Lecare", „In Prüfung" |
| `PruefungFinanzen/Einkauf/ITSec/ArbeitsrechtlicheFragen` | Choice/Text | „In Prüfung" (leer = nicht benötigt; `@empty(PruefungITSec)` = keine ITSec-Prüfung) |
| `NeuerStatus`, `NeuerKommentar` | Ja/Nein | Steuerflags Modified-Trigger (Flow resettet) |
| `KommentarVon` | Multi-Choice | Legal, Finance, Genehmiger, Indirect Sourcing, Antragsteller, AntragstellerAP, LegalAP, ITSec, AntragstellerITSec, Kein neuer Kommentar |
| `AbgelehntDurch` | Choice | LegalAP, Genehmiger, Finance, Indirect Sourcing, Legal, ITSec |
| `Einsatz_Software` | Ja/Nein | ITSec-Auslöser (B2) |
| weitere | | AufwandsbezogeneVerguetung, Whitelist, Generalfreigabe, PruefenderRechtsanwalt, VerantwortlicherMitarbeiter, Genehmiger, Fachbereich, Vertragsart |

## 4. Connection References

| Logical Name | Connector | genutzt von |
|---|---|---|
| `hse_EFABOSharePoint` | SharePoint | fast alle Flows |
| `hse_sharedoffice365_44daf` | Office 365 Outlook | EzP-Child, Druck-Flows, Geänderter Vertrag, Neuer Antrag |
| `hse_sharedonedriveforbusiness_96b42` | OneDrive for Business | Druck-Flows (PDF-Konvertierung) |
| `hse_sharedteams_30bd2` | Teams | Geänderter Vertrag (+DEVTEST) |
| `hse_sharedsharepointonline_f07a0` | SharePoint | nur DEVTEST-Flow |
| `hse_sharedsharepointonline_873bd` | SharePoint | kein Flow (vermutlich Canvas-App) |

Service-Account-Umstellung (`desvc.efabo@hse.com`): alle 6 Referenzen auf SA-Connections binden; meiste Aktionen `embedded`. **Ausnahme invoker:** CheckifuserinSharePointGroup + Druck-Flows (aktuell) – bei Auto-Druck zwingend auf embedded/SA umstellen (OneDrive-Konvertierung läuft dann im SA-OneDrive).

## 5. Environment-Variablen (Auszug, DEV-Defaults)

| Name | Typ | Zweck / Default |
|---|---|---|
| `hse_shared_sharepointonline_f3b34deb…` | Datenquelle | EFABO Site – `…/sites/EFABO_DEV` |
| `hse_shared_sharepointonline_fcca5627…` | Datenquelle | Liste EFABO Anträge |
| `new_shared_sharepointonline_428af0df…` | Datenquelle | Liste Arbeitsrechtliche Freigabe |
| `hse_ITSecurityFreigabe` | Datenquelle | Liste IT-Security Freigabe |
| `hse_EFABO_Document_Library` | Datenquelle | Doc Library – **einzige Var mit `environmentvariablevalues.json` im Paket (Wert reist mit!)** |
| `hse_AppID` | String | Canvas-App-ID für Mail-Deep-Links |
| `hse_EFABOTesting` | Boolean | „TEST: "-Mailpräfix – DEV-Default `true` (**PROD: false!**) |
| `hse_RoleDefIdErstellen` / `hse_RoleDefIdPruefen` | String | 1073741926 / 1073741927 (Workermanagement-Read `1073741826` **hartkodiert**) |
| `hse_EFABO_Gruppe_*ID` | String | SP-Principal-IDs: Einkauf 12, Finance 14, Recht 15, SB-Recht 16, Workermanagement 40, ITSec 45 |
| `hse_TeamsID*` / `hse_TeamsChannelID*` | String | **tot** – Teams-Aktionen in „Geänderter Vertrag" haben groupId/channelId hartkodiert |

Env-Var-Definitionen sind **nicht** RootComponents (reisen nur als Abhängigkeit mit).

## 6. Solution-Metadaten

- UniqueName `EFABO`, **v1.0.1.5**, unmanaged, Sprache 1031. Publisher `HSE`, Prefix `hse`.
- RootComponents: 13× Workflow (inkl. Testartefakte „Geänderter Vertrag - DEV TEST", „Test FLow Anlage") + 1× CanvasApp. MissingDependencies leer.
- **Keine Dataverse-Tabellen** – SharePoint-Datenhaltung bestätigt.
- ⚠️ Kern-Flows „Neuer Antrag" + „Geänderter Vertrag" (+DEVTEST) im Export **Draft/deaktiviert** – Import liefert Automatiken deaktiviert aus.
- ⚠️ DEVTEST = Duplikat mit identischer Trigger-Condition → **Doppel-Feuer-Risiko** wenn beide aktiv.
- ⚠️ Hartkodierte Teams-IDs + Read-RoleDefId entziehen sich Env-Var-Steuerung → nicht umgebungsfähig.
