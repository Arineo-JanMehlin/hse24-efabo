# ALM-Drift DEV ↔ Prod (Stand 21.08.2026)

Vollständiger Abgleich aller EFABO-Flows zwischen der Prod-Umgebung (inkl. unmanaged Layer)
und dem DEV-Stand. Anlass: Untersuchung des unmanaged Layers auf `NeuerAntrag` während der
Störung aus [05-bug-service-account-verbindung.md](05-bug-service-account-verbindung.md).

## Umgebungen

| | Prod | DEV |
|---|---|---|
| Environment ID | `65b252e4-80e4-ee82-bb99-e3914b234a52` | `066791b7-04a5-43a6-b01d-b1d6b971e869` |
| Org URL | `https://orgdb2ff17e.crm4.dynamics.com/` | `https://hsedevelopment.crm4.dynamics.com/` |
| Solution EFABO | **1.0.1.5 managed** | **1.0.2.1 unmanaged** |

Prod hinkt also eine Minor-Version hinterher. Das Repo unter [src/](../src/) ist ein exaktes
Abbild von DEV — Vergleich aller 13 Solution-Flows: 0 Abweichungen.

### Methodik

Die Prod-Definitionen wurden live über die virtuelle Tabelle `msdyn_componentlayer`
ausgelesen (`pac env fetch`), nicht aus einem Export. Der Solution-Export aus DEV schlägt
mit dem Service Account fehl:

```
Error: user with id 6211fcd9-7248-ec11-8c62-000d3a4426c1 does not have ReadAccess right(s)
for record with id cca578b7-c479-ee11-8179-6045bda1ee11 of entity Verbindungsreferenz
```

Der SA `desvc.efabo@hse.com` darf eine Connection Reference in DEV nicht lesen, die einem
anderen Benutzer (`5e641969-e644-ec11-8c60-6045bd8d2ce1`) gehört. Die DEV-Definitionen
wurden deshalb direkt aus der `workflow`-Tabelle gelesen. **Offen:** Leserechte des SA auf
diesen Datensatz, sonst ist kein Solution-Export aus DEV mit dem SA möglich.

## Unmanaged Layer in Prod

Geprüft gegen `msdyn_componentlayer`, nicht gegen `solutioncomponent` — eine Registrierung in
der Active-Solution bedeutet **keinen** Layer. 5 von 44 Komponenten tragen tatsächlich einen:

| Komponente | Layer | unmanaged | overwritetime managed |
|---|---|---|---|
| Erfassungsbogen Verträge (Canvas-App) | 2 | **ja** | – |
| NeuerAntrag | 2 | **ja** | 12.08.2025 20:31 |
| CheckifuserinSharePointGroup | 2 | **ja** | 04.05.2025 22:12 |
| CreateHTML_PDF_AF | 2 | **ja** | 05.05.2025 08:26 |
| CreateHTML_PDF_ITSec | 2 | **ja** | 05.05.2025 08:25 |
| Elementarchivieren | 1 | nein | |
| GenderterVertrag | 1 | nein | |
| ChildEFABOBerechtigungenndern | 1 | nein | |
| ChildEFABOEntwurfzuPrfung | 1 | nein | |
| ParentEFABOBerechtigungenndern | 1 | nein | |
| ParentEFABOEntwurfzuPrfung | 1 | nein | |
| EFABOEntwurferstellen | 1 | nein | |
| GenderterVertrag-DEVTEST | 1 | nein | |
| TestFLowAnlage | 1 | nein | |

Der unmanaged Layer gewinnt gegen den managed Layer. Ein Solution-Import überschreibt diese
5 Komponenten **nicht** — er landet im managed Layer darunter und ändert am laufenden
Verhalten nichts.

## Befunde

### 1. `CreateHTML_PDF_AF` — DEV war falsch, Prod korrekt → **in DEV nachgezogen**

Der Flow wurde in DEV als Kopie von `CreateHTML_PDF_ITSec` angelegt und nur teilweise
umgestellt. Eine Aktion `Get_item_-_ITSec_Fragen` blieb stehen, dazu der PDF-Dateiname:

| | DEV (vorher) | Prod |
|---|---|---|
| Kette | `Get_item_-_Arbeitsrechtliche_Fragen` → `Get_item_-_ITSec_Fragen` → `Compose` | `Get_item_-_Arbeitsrechtliche_Fragen` → `Compose` |
| PDF-Name | `Export IT Security Fragen für EFABO (…ITSec…)` | `Arbeitsrechtliche Fragen für EFABO (…AF…)` |

`Compose_-_HTML_structure` referenziert in beiden Ständen ausschließlich
`Get_item_-_Arbeitsrechtliche_Fragen`. Der ITSec-Abruf war in DEV toter Ballast, hat aber
über den Dateinamen die falsche PDF-Bezeichnung erzeugt und pro Lauf einen überflüssigen
SharePoint-Call verursacht.

**Erledigt im Repo** — Prod-Stand übernommen: Aktion entfernt, `runAfter` auf
`Get_item_-_Arbeitsrechtliche_Fragen`, PDF-Name korrigiert.

### 2. `NeuerAntrag` — ParseJson-Schemas, in beide Richtungen kaputt

In beiden Umgebungen wurde je einmal die komplette Aktionsdefinition ins Schema-Feld
eingefügt statt des Schemas. Jeweils die andere Seite ist korrekt:

| Aktion | DEV vorher | Prod | korrekt |
|---|---|---|---|
| `Parse_Arbeitsrechtliche_Prüfung` | 2 Props `content,schema` | 46 Props flach | Prod |
| `Parse_ITSec_Prüfung` | 54 Props flach | 2 Props `content,schema` | DEV |

Laufzeitkritisch ist keiner der beiden: alle Downstream-Zugriffe lauten
`body('Parse_…')?['<Feld>']` und sind auf beiden Seiten identisch; ParseJson ohne `required`
lässt nicht passende Payloads durch. Kaputt ist die Dynamic-Content-Auswahl im Designer.

**Erledigt im Repo** — `Parse_Arbeitsrechtliche_Prüfung` auf den Prod-Stand (46 Properties)
gesetzt. Das korrigierte Schema liegt zum Einfügen im Designer separat unter
[06-schema-Parse_Arbeitsrechtliche_Pruefung.json](06-schema-Parse_Arbeitsrechtliche_Pruefung.json).

`Parse_ITSec_Prüfung` bleibt in DEV unangetastet — dort ist DEV die gesunde Seite. Das ist
ein **Prod-Defekt**, der mit dem nächsten Deployment mitkommt.

### 3. `ParentEFABOBerechtigungenndern` — Prod-Defekt, DEV bereits korrekt

Der Parent gibt zwei Parameter vertauscht an den Child-Flow weiter.

Vertrag des Child-Flows (`ChildEFABOBerechtigungenndern`, in beiden Ständen identisch):
`text` = NewUserMail · `text_1` = RemoveUserMail · `text_2` = Role · `number` = EfaboID

Belegung im Parent (`InitializeVariable`, in beiden Ständen identisch):
`Variableinitialisieren_Wert` = **Role** · `Variableinitialisieren2_Wert` = **NewUserMail**

| | `text` (NewUserMail) | `text_2` (Role) |
|---|---|---|
| DEV | `Variableinitialisieren2_Wert` ✔ | `Variableinitialisieren_Wert` ✔ |
| **Prod** | `Variableinitialisieren_Wert` ✘ Role | `Variableinitialisieren2_Wert` ✘ Mail |

Auswirkung in Prod: der Child ruft `_api/web/SiteUsers/getbyemail('Genehmiger')` auf — kein
gültiger Wert. Zusätzlich vergleicht `if(equals(variables('Role'),'Genehmiger'), …)` eine
E-Mail-Adresse gegen `'Genehmiger'`, wird also nie wahr und vergibt immer
`Role Def Id Erstellen` statt `Pruefen`.

Der Aufruf kommt aus der Canvas-App (Trigger `kind: PowerApp`). **Nicht nach DEV
nachziehen** — DEV ist die korrigierte Seite. Fix erreicht Prod erst mit dem Deployment.

### 4. DEV ist Prod voraus — nicht nachziehen, sondern deployen

| Was | Wo |
|---|---|
| `EFABO Druck und Versand` (`3f8e2d4a-…`), Status **Entwurf** | nur DEV |
| `[Parent] EFABO Druck und Versand` (`7c4b9e1d-…`), Status **Entwurf** | nur DEV |
| `GenderterVertrag`: 2 Child-Flow-Aufrufe `Druck Abgelehnt` / `Druck Abgeschlossen` | nur DEV |
| `ChildEFABOEntwurfzuPrfung`: kompletter Zweig `Bedingung - Einsatz Software wird benötigt` mit 14 Aktionen | nur DEV |

Beide Druck-Flows stehen in DEV auf **Entwurf** und sind nicht Teil der Prod-Solution.

### 5. Reines Rauschen — kein Handlungsbedarf

`inputs.authentication` (`@parameters('$authentication')`), `operationMetadataId`,
`connectionName`-Suffixe (`shared_sharepointonline_1` vs `_3`) und die in `clientdata`
eingebetteten Umgebungsvariablen-Defaults (`…/sites/EFABO` vs `…/sites/EFABO_DEV`).
Betrifft `CheckifuserinSharePointGroup` (2 Diffs) und `CreateHTML_PDF_ITSec` (9 Diffs)
vollständig.

Vollständig identisch: `Elementarchivieren`, `ChildEFABOBerechtigungenndern`,
`ParentEFABOEntwurfzuPrfung`, `EFABOEntwurferstellen`, `GenderterVertrag-DEVTEST`,
`TestFLowAnlage`.

## Umgebungsvariablen Prod vs DEV

Beim Deployment zwingend prüfen — zwei Werte sind über Kreuz belegt:

| Variable | Prod | DEV |
|---|---|---|
| `Role Def Id Pruefen` | **1073741924** | 1073741927 |
| `Role Def Id Erstellen` | **1073741927** | 1073741926 |
| `EFABO Site Collection` | `…/sites/EFABO` | `…/sites/EFABO_DEV` |
| `EFABO Testing` | `false` | `true` |
| `hse_EFABO_Gruppe_EinkaufID` | 22 | 12 |
| `hse_EFABO_Gruppe_FinanceID` | 20 | 14 |
| `hse_EFABO_Gruppe_ITSecID` | 367 | 45 |
| `hse_EFABO_Gruppe_RechtID` | 21 | 15 |
| `hse_EFABO_Gruppe_SachbearbeiterRechtID` | 18 | 16 |
| `hse_EFABO_Gruppe_WorkermanagementID` | 182 | 40 |

*(Zuordnung 21.08. abends gegen die `environmentvariablevalue`-Tabellen beider Umgebungen
verifiziert — die frühere Sammelzeile hatte die Werte teils falsch zugeordnet.)*

`1073741927` bedeutet in Prod *Erstellen*, in DEV *Pruefen*. Wer den unmanaged Layer auf
`NeuerAntrag` entfernt, fällt auf den managed Stand samt DEV-Defaults zurück und vergibt
danach systematisch falsche Berechtigungsstufen.

## Nachtrag 21.08.2026: Fixes in DEV angewendet, Connection References konsolidiert

Beide Fixes wurden per Dataverse Web API (`PATCH workflows(…).clientdata`, Konto
`desvc.efabo@hse.com` via `az` CLI) direkt in DEV eingespielt. Zusätzlich auf Wunsch von
Jan: alle SharePoint-Bindungen der Flows auf die Connection Reference
**`hse_sharedsharepointonline_f07a0`** umgestellt.

### Connection-Reference-Inventur DEV

Die Solution enthält 6 CRs, der SA sieht nur 5:

| Logical Name | Anzeige | Zustand |
|---|---|---|
| `hse_sharedsharepointonline_f07a0` | SharePoint EFABO | **neuer Standard**; Verbindung war tot/fremd → auf SA-Verbindung `66248777e40c4ad79a33674db7ca08c1` umgebunden |
| `hse_EFABOSharePoint` | EFABO SharePoint | alt, nach Umstellung nur noch von 2 Entwurfs-Flows referenziert |
| `hse_sharedsharepointonline_873bd` | SharePoint EFABO-873bd | **= `cca578b7-…`, der Export-Blocker.** Für SA unsichtbar (gehört `5e641969-…`), von keinem Flow genutzt. 21.08. per `RemoveSolutionComponent` aus der Solution entfernt (Record besteht weiter) — Export-Blocker damit behoben |
| `hse_sharedoffice365_44daf` / `hse_sharedonedriveforbusiness_96b42` / `hse_sharedteams_30bd2` | | ok, an lebende SA-Verbindungen gebunden |

### Ergebnis der Umstellung (11 Flows geplant)

9 erfolgreich (aktivierte Flows per Deaktivieren → Patch → Reaktivieren; die Reaktivierung
validiert die Verbindung — alle 8 kamen sauber zurück auf Aktiviert):
`NeuerAntrag` (inkl. Schema-Fix), `CheckifuserinSharePointGroup`,
`ChildEFABOBerechtigungenndern`, `ChildEFABOEntwurfzuPrfung`, `CreateHTML_PDF_AF`
(inkl. Fix 1), `CreateHTML_PDF_ITSec`, `EFABOEntwurferstellen`, `Elementarchivieren`,
`TestFLowAnlage`. `GenderterVertrag-DEVTEST` stand schon auf f07a0.

2 fehlgeschlagen, beide Entwurf, beide Druck-Feature:

- `GenderterVertrag` — `ChildFlowNeverPublished`: referenziert den nie veröffentlichten
  Child `EFABO Druck und Versand` (`3f8e2d4a-…`); der Flow-Service lehnt jeden Save ab,
  bis der Child veröffentlicht ist. Alter Stand (CR `hse_EFABOSharePoint`) bleibt.
- `EFABODruckundVersand` — `does not have WriteAccess … of entity Prozess`: Datensatz
  gehört nicht dem SA. Gleiche Rechtelücke wie beim Export.

Repo wurde für diese 2 zurückgestellt und spiegelt DEV wieder exakt (verifiziert:
0 strukturelle Diffs über alle 11 Flows; einzige tolerierte Abweichung ist
`properties.templateName: null`, das Dataverse beim Speichern verwirft).

### Nachtrag 21.08. (nachmittags): Druck-Flows unter SA neu angelegt, alte CR gelöscht

Die 2 offenen Flows gehörten Jans persönlichem Account — der SA hat auf fremde
`Prozess`-Datensätze weder Write- noch DeleteAccess. Lösung: Jan hat beide Flows im
Maker-Portal selbst gelöscht (dafür musste er in `GenderterVertrag` vorher die beiden
„Untergeordneten Flow ausführen"-Aktionen entfernen, sonst blockierte die
Child-Abhängigkeit die Löschung). Danach per Web API:

1. Beide Flows als Upsert (`PATCH` + `If-None-Match: *` + `MSCRM.SolutionUniqueName:
   EFABO`) **mit identischen GUIDs** unter dem SA neu angelegt — `3f8e2d4a-…` (Child)
   und `7c4b9e1d-…` (Parent); Referenzen aus `GenderterVertrag`/Parent bleiben gültig.
   Child direkt mit CR f07a0.
2. Child einmal aktiviert → hebt `ChildFlowNeverPublished` dauerhaft auf.
3. `GenderterVertrag` mit vollständiger Original-Definition gepatcht: stellt die von
   Jan entfernten Child-Aktionen wieder her **und** stellt auf f07a0 um.
4. Child zurück auf Entwurf (Originalzustand).

Endzustand verifiziert: alle 3 Flows Owner `DESVC EFABO`, in Solution EFABO,
state 0/1 (Entwurf), CR f07a0. Kein Flow referenziert mehr `hse_EFABOSharePoint`
→ CR **gelöscht** (`10809114-…`, 0 Referenzen, 0 Dependencies, DELETE + 404 verifiziert).

Damit sind alle 11 Flows auf `hse_sharedsharepointonline_f07a0` konsolidiert und
sämtliche Flows der Solution gehören dem SA.

Zusätzlich wurde `hse_sharedsharepointonline_873bd` (`cca578b7-…`, der Export-Blocker)
per `RemoveSolutionComponent` aus der Solution entfernt — geht ohne Leserechte auf den
Record, weil es eine Solution-Operation ist; der (für den SA unsichtbare) CR-Record
selbst besteht weiter. `pac solution export` danach erfolgreich getestet. Das dafür
angedachte Delegationsticket an die HSE-IT ist damit hinfällig; auf die übrigen
Ticket-Punkte (CA-Ausnahme, Service Principal) wird laut Jan verzichtet.

## Nachtrag 21.08. (abends): Erneuter Vollabgleich DEV ↔ Prod

Nach Abschluss aller DEV-Arbeiten (CR-Konsolidierung, Druck-Flows, 873bd) wurde der
Abgleich komplett wiederholt. Methodik diesmal: DEV per `pac solution export`
(funktioniert jetzt), Prod-Effektivstand (= inkl. unmanaged Layer) per `pac org fetch`
direkt aus der `workflow`-Tabelle; struktureller JSON-Diff mit Normalisierung
(`operationMetadataId`, `templateName`, `$authentication`, Connection-Suffixe,
CR-Logical-Names, eingebettete Env-Var-Defaults). Skripte und Rohdaten im
Session-Scratchpad unter `cmp\`.

### Ergebnis pro Flow (nur echte Diffs)

| Flow | Diffs | Bewertung |
|---|---|---|
| `NeuerAntrag` | 56 | alle im kaputten `Parse_ITSec_Prüfung`-Schema (Prod-Defekt 2., unverändert). `Parse_Arbeitsrechtliche_Prüfung` nach DEV-Fix jetzt identisch ✔ |
| `GenderterVertrag` | 2 | die 2 Druck-Child-Aufrufe, nur DEV (Befund 4., DEV voraus) |
| `ChildEFABOEntwurfzuPrfung` | 8 | Software-Zweig + Verkabelung, nur DEV (Befund 4., DEV voraus) |
| `ParentEFABOBerechtigungenndern` | 2 | vertauschte `text`/`text_2` in Prod (Prod-Defekt 3., unverändert) |
| `CreateHTML_PDF_AF` | 8 | 6× Binding (s. u.) + **neuer DEV-Defekt 6.** |
| `CreateHTML_PDF_ITSec` | 7 | 6× Binding + 1 kosmetisches `additionalProperties` — inhaltlich identisch |
| übrige 7 Flows | 0 | identisch |

Binding-Diffs (kein Inhalt): die Prod-Flows mit unmanaged Layer tragen `runtimeSource:
embedded` mit fest verdrahteten Connections, DEV `invoker` + Connection References
(jetzt f07a0). Beim Deployment normal auflösbar, aber: **f07a0 existiert in Prod noch
nicht** und muss beim Import auf eine gültige Prod-Verbindung gebunden werden; die aus
der DEV-Solution entfernten CRs (`hse_EFABOSharePoint`, 873bd) verschwinden in Prod nur
bei einem Upgrade-Import (nicht bei Update).

### 6. `CreateHTML_PDF_AF` — neuer DEV-Defekt: fehlende Parameter-Deklaration

In DEV verwendet `Get_item_-_Arbeitsrechtliche_Fragen` als `table`
`@parameters('Arbeitsrechtliche Freigabe (new_shared_sharepointonline_428af0dfff…)')`,
aber `definition.parameters` deklariert diesen Parameter **nicht** — deklariert ist
stattdessen das ungenutzte `IT-Security Freigabe (hse_ITSecurityFreigabe)` (Altlast der
ITSec-Kopie). Prod deklariert korrekt. Vermutlich beim Fix 1. (Web-API-Patch) die
Deklaration nicht mitgezogen; Speichern/Aktivieren validiert das nicht, zur Laufzeit
fällt die Aktion aber mit „parameter not defined" um. **Fix in DEV nötig:** Deklaration
ergänzen (`defaultValue` = DEV-Wert `312ce00d-e31d-4435-8e4c-63e76f0e6dfa`), ungenutzte
IT-Security-Deklaration entfernen.

### Statecode-Abweichungen

| Flow | Prod | DEV |
|---|---|---|
| `NeuerAntrag` | Aktiviert | Entwurf |
| `GenderterVertrag` | Aktiviert | Entwurf |
| `Elementarchivieren` | **Entwurf** | Aktiviert |

`Elementarchivieren` ist in Prod deaktiviert — Archivierung läuft dort aktuell nicht
(klären, ob Absicht).

### Layer & Canvas-App

Unmanaged Layer in Prod unverändert (dieselben 5 Komponenten, gleiche overwritetimes).
Canvas-App „Erfassungsbogen Verträge": Prod-`appversion` **24.03.2026** (nach dem
Import 1.0.1.5 direkt in Prod editiert), DEV 18.08.2026. Inhaltlicher Vergleich weiter
offen.

## Offene Punkte

- [x] ~~Beide Fixes aus 1. und 2. in der DEV-Umgebung anwenden~~ → erledigt 21.08. per Web API
- [x] ~~CR-Umstellung auf f07a0 für `GenderterVertrag` + `EFABODruckundVersand`~~ →
      erledigt 21.08.: Flows unter SA mit identischen GUIDs neu angelegt, Child einmal
      veröffentlicht, alle 11 Flows auf f07a0
- [x] ~~`hse_sharedsharepointonline_873bd` (= Export-Blocker `cca578b7-…`) aus der Solution
      entfernen oder Besitz auf SA übertragen~~ → erledigt 21.08. per
      `RemoveSolutionComponent` (kein Admin nötig), Export erfolgreich getestet
- [x] ~~`hse_EFABOSharePoint` nach vollständiger Umstellung aus der Solution nehmen~~ →
      erledigt 21.08.: CR gelöscht (0 Referenzen, 0 Dependencies)
- [x] ~~**DEV-Defekt 6. fixen:** `CreateHTML_PDF_AF` — Parameter-Deklaration
      `Arbeitsrechtliche Freigabe (…428af…)` ergänzen, ungenutzte
      `IT-Security Freigabe`-Deklaration entfernen~~ → erledigt 21.08. abends per
      Web-API-Patch (Deaktivieren → clientdata → Reaktivieren, validiert), Repo nachgezogen
- [ ] Prod-Defekte 2. (`Parse_ITSec_Prüfung`) und 3. (vertauschte Parameter) deployen
- [ ] Klären: `Elementarchivieren` in Prod auf Entwurf — Absicht?
- [ ] Unmanaged Layer in Prod auflösen — pro Komponente Prod-Stand nach DEV, deployen,
      dann Layer entfernen. Nicht vor dem CA-Fix und nicht ohne Abgleich der
      Umgebungsvariablen.
- [ ] Canvas-App „Erfassungsbogen Verträge": unmanaged Layer noch nicht inhaltlich
      verglichen
