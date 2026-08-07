---
tags:
  - kunde/hse24
  - power-automate
  - todo
date: 2026-06-03
status: in-progress
---

# HSE24

## Offene Aufgaben

- [ ] Flows zusammenführen und Trigger anpassen
- [ ] Zusammensetzung HTML Bau
- [ ] Anpassung Drucklayout (schöner gestalten)
- [ ] Drucklogik auf EFABO Hauptformular erweitern
- [ ] Zusammenführung Logik aller bisherigen Flows und Triggeränderung





Das ursprüngliche Angebot:

|     |                                                                                                               |      |     |
| --- | ------------------------------------------------------------------------------------------------------------- | ---- | --- |
| 1   | Anpassung Drucklayout                                                                                         | 2,00 | CS  |
| 2   | Drucklogik für EFABO Hauptformular                                                                            | 8,00 | CS  |
| 3   | Neue Workflow Logik für Automtischen Druck und Benachrichtigung bei EFABO Status abgeschlossen oder abgelehnt | 3,00 | CS  |
| 4   | Entfernen Druckbuttons in App                                                                                 | 0,50 | CS  |
| 5   | Test und GoLive                                                                                               | 1,50 | CS  |
| 6   | **Projektmanagement und Abstimmungstätigkeiten**  <br>Demo-/Abstimmungstermine, Kommunikation, usw.           | 1,00 | CS  |

**Anpassung Drucklayout**

- sie waren nicht happy, wie der ausgedruckte PDF aussieht
- es ging primär um Formatierungen (2 Spaltig, Umbrüche, etc.)
- **Am besten das Thema im Meeting ansprechen und dir Feedback geben lassen**

**Drucklogik für EFABO Hauptformular und neue Workflow Logik**

- Mit der Druckfunktion können sie sich ja beim Abschluss, der Prüfung die ausgefüllten Werte ausdrucken und irgendwo Auditsicher ablegen
- Die Ursprüngliche Idee war, dass die Druck-Buttons aus den Unterformularen verschwinden und dann nur noch auf dem Hauptformular ein Knopf liegt
    - der Button startet alle Druckflows und schickt an die entsprechenden Leute Mails oder legt das irgendwo ab
    - bisher ging die Mail immer an die Person, die den Button drückt
- Wir hatten dann die Idee so geändert, dass einfach autom. mit Abschluss der Prüfung der Haupt Workflow einfach die Druck-Flows als Subflows autom. aufruft und die gar keine Buttons mehr drücken müssen
    - sie hatten dann Panik bekommen, dass wenn die Buttons weg sind, sie das bei Fehlern nicht selbst nochmal starten können

**Entfernen Druckbuttons in App**

- der Punkt fällt dann weg Sie hatten zusätzlich als Bug mal gemeldet, dass die Verlängerung des EFABOs nicht mehr funktioniert. War so eine Extra-Funktion mit einem Button, der bei bestimmten Szenarien eingeblendet wird. Ging dann wohl einfach so wieder. Mir fehlt der Überblick, was sonst ihre aktuellen Probleme sind. Ich hatte massive Probleme wegen den Druck-Buttons auf Prod. Ging für manche User und manche nicht, hatte dann ein unmanaged Layer um den Flow nochmal in der App zu Entfernen und wieder hinzuzufügen. Mir wäre es am liebsten, wenn die beiden Buttons weg sind und es einen neuen mit einem neuen Flow gibt. Weiß nur nicht, ob das die Zeit zu lässt. Aufwände sind in h. Wäre aber sicher sinnvoll, wenn du Claude mal kurz schauen lässt, ob er noch irgendwelche Änderungen in Flows/Apps mit unmanaged Layer findet. Ich hab versucht, alles was ich in eine Unmanaged Layer gemacht habe, direkt auch noch in DEV umzusetzen. Ging nur manchmal kein Update so schnell. 