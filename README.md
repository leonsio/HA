# Zendure/Solarman Nulleinspeisung mit zwei Speichern

README-Stand: v17 Dokumentation  
Blueprint-Logik: v17

Diese README beschreibt die wesentlichen Eingabeparameter, die automatisch erkannten Zendure-/Solarman-/Shelly-Entitäten und die Regeln der Lade-, Entlade- und Speicher-zu-Speicher-Strategie.

Der Blueprint regelt zwei Zendure-Speicher aus der Zendure Home Assistant Integration zusammen mit einem über Solarman angebundenen Wechselrichter. Ziel ist eine möglichst geringe Netzeinspeisung, ohne unnötig Energie zu verlieren, und eine sinnvolle Verteilung zwischen beiden Speichern.

---

## 1. Grundprinzip

Die Automation läuft jede Sekunde.

Der Regelzyklus arbeitet in dieser Reihenfolge:

1. Geräte und Entitäten automatisch erkennen.
2. Kritische Entitäten und Sensorwerte prüfen.
3. Off-Grid-Port desjenigen Speichers schützen, an dessen Off-Grid-Port der Wechselrichter angeschlossen ist.
4. Netzleistung, Speicherzustände, Kapazitäten und technische Leistungsgrenzen einlesen.
5. Betriebszustand bestimmen: Notladung, Speicher-zu-Speicher-Ausgleich, Überschussladung, Entladung gegen Netzbezug oder Leerlauf.
6. Zielwerte für beide Speicher berechnen.
7. Solarman-Wechselrichterleistung als festen Ziel-Prozentwert berechnen.
8. Neue Werte nur schreiben, wenn sie sich relevant vom aktuellen Zustand unterscheiden.

Wichtig: Die Speicher werden über Zendure-AC-Modi gesteuert. Laden verwendet fest den AC-Modus `input`, Entladen verwendet fest den AC-Modus `output`. Der Off-Grid-Port wird mit `off` ausgeschaltet und mit `normal` eingeschaltet.

---

## 2. Automatisch erkannte Entitäten

In der Blueprint-Maske werden keine einzelnen Zendure-Entitäten mehr ausgewählt. Stattdessen wählst du Geräte aus.

### Speicher 1 und Speicher 2

Für beide Speicher müssen physische Zendure-Geräte aus der Integration `zendure_ha` ausgewählt werden. Der Blueprint sucht innerhalb des jeweiligen Geräts automatisch unter anderem folgende Entitäten:

| Zweck | Typische Zendure-HA-Suffixe |
|---|---|
| SoC / Ladezustand | `*_electric_level`, `*_soc_level`, `*_battery_level` |
| aktuelle Batterieladeleistung | `*_output_pack_power`, `*_battery_input_power`, `*_charge_power` |
| optionale Batterieentladeleistung | `*_pack_input_power`, `*_battery_output_power`, `*_discharge_power` |
| Gesamtkapazität | `*_total_kwh` |
| verfügbare Energie | `*_available_kwh` |
| Off-Grid-Ladeleistung | `*_grid_off_power`, `*_gridoffpower`, `*_off_grid_power`, `*_offgrid_power` |
| Solar-Ladeleistung | `*_solar_input_power`, `*_solarinputpower`, `*_pv_input_power` |
| AC-Modus | `*_ac_mode`, `*_acmode` |
| AC-Eingangsgrenze | `*_input_limit`, `*_ac_input_limit`, `*_inputlimit` |
| AC-Ausgangsgrenze | `*_output_limit`, `*_ac_output_limit`, `*_outputlimit` |
| maximale Entlade-/Transferleistung | `*_inverse_max_power`, `*_inversemaxpower` |
| Off-Grid-Port | `*_grid_off_mode`, `*_off_grid_mode`, `*_offgrid_mode` oder passender Switch |

Die Gesamtkapazität wird aus `*_total_kwh` gelesen. Die verfügbare Energie wird aus `*_available_kwh` gelesen. Die aktuelle Ladeleistung für Entlade- und Transferentscheidungen wird aus den Ladequellen `*_grid_off_power` und `*_solar_input_power` gebildet. `*_grid_off_power` wird dabei absolut genommen, weil Zendure diesen Wert je nach Richtung auch negativ liefern kann. Als Schutz gegen Doppelzählung nutzt der Blueprint für die Regelung den größeren Wert aus bisheriger Pack-Ladeleistung und Summe aus Off-Grid- plus Solar-Ladeleistung:

```text
charge_for_distribution = max(
  output_pack_power,
  abs(grid_off_power) + max(0, solar_input_power)
)
```

Die maximale Entlade- und Transferleistung wird aus `*_inverse_max_power` gelesen und zusätzlich durch das `max`-Attribut der Zendure-`output_limit`-Number begrenzt, falls dieses kleiner ist.

### Netzanschluss

Für den Netzanschluss wird ein Shelly-Gerät ausgewählt. Der Blueprint sucht bevorzugt eine Gesamtleistungs-Entität wie:

- `*_total_active_power`
- `*_total_power`
- `*_active_power_total`
- `*_power_total`

Erwartete Vorzeichenlogik nach optionaler Invertierung:

- positiver Wert = Netzbezug
- negativer Wert = Einspeisung

### Wechselrichter

Für den Wechselrichter wird ein Solarman-Gerät ausgewählt. Darin wird die Number-Entität `*_active_power_regulation` gesucht. Diese Entität wird direkt mit einem Prozentwert von 0 bis 100 beschrieben.

Ab v17 wird die Drosselung nicht mehr aus Hausverbrauch, Restüberschuss oder Rampenschritten berechnet. Wenn die Drosselbedingung erfüllt ist, schreibt der Blueprint den festen konfigurierten Prozentwert `Wechselrichter - fester Drosselwert`. Default: 50 %.

Beispiel: Bei `inverter_max_w = 2000 W` entspricht ein Solarman-Wert von 50 Prozent rechnerisch einer Wechselrichterleistung von 1000 W. Diese Watt-Umrechnung dient nur noch zur Diagnose; an Solarman wird der Prozentwert geschrieben.

---

## 3. Wichtige Parameter

### Netz- und Regelparameter

| Parameter | Bedeutung |
|---|---|
| `Netzleistung Vorzeichen invertieren` | Dreht das Vorzeichen des Shelly-Messwerts, falls dein Zähler Einspeisung und Bezug umgekehrt meldet. |
| `Totband um Nulleinspeisung` | Kleine Leistungsabweichungen innerhalb dieses Bereichs werden ignoriert, damit die Regelung nicht bei jedem Watt nachregelt. |
| `Mindest-Änderung für neue Leistungsvorgabe` | Ein neuer Zendure-Wert wird nur geschrieben, wenn die Differenz mindestens diesen Wert erreicht. Reduziert unnötige Schreibvorgänge. |
| `Maximale Ladeleistung über das Hausnetz` | Begrenzt normale Überschussladung und Netz-Notladung. Dieser Wert begrenzt nicht die normale Entladung und nicht den Speicher-zu-Speicher-Transfer. Default: 1000 W. |

### Speicher-SoC-Parameter

| Parameter | Bedeutung |
|---|---|
| `Speicher gilt als voll ab SoC` | Ab diesem SoC wird ein Speicher als voll betrachtet. Default: 98 %. |
| `Entlade-Reserve SoC` | Unterer SoC-Wert für normale Entladung. Speicher mit SoC kleiner/gleich diesem Wert werden nicht mehr zur Hausversorgung entladen. Default: 15 %. |
| `Netz-Nachladen starten unter SoC` | Unterhalb dieses SoC wird Notladung aktiviert. Default: 10 %. |
| `Netz-Nachladen fortsetzen bis SoC` | Eine bereits gestartete Notladung läuft bis zu diesem SoC weiter. Default: 20 %. |
| `Reserve für gegenseitige Notaufladung über SoC-Limit` | Ein Speicher darf einen anderen bei Notladung nur versorgen, wenn er mindestens diesen Abstand oberhalb des Notlade-Startwerts liegt und zugleich höheren SoC hat. Default: 20 Prozentpunkte. |
| `Mindest-SoC-Vorsprung des abgebenden Speichers` | Speicher-zu-Speicher-Ausgleich ist nur erlaubt, wenn der abgebende Speicher mindestens diesen SoC-Vorsprung hat. Default: 0 %. |

### Speicher-zu-Speicher-Parameter

| Parameter | Bedeutung |
|---|---|
| `Tolerierter Quell-Überschuss vor Speicher-Transfer` | Bei nicht vollem abgebendem Speicher wird nur der quellbezogene Überschuss oberhalb dieses Werts an den anderen Speicher weitergegeben. Default: 1000 W. |
| `Mindestleistung für Speicher-Ausgleich` | Unterhalb dieser final berechneten Transferleistung wird kein zusätzlicher Speicher-zu-Speicher-Ausgleich gestartet. Default: 50 W. |

### Wechselrichterparameter

| Parameter | Bedeutung |
|---|---|
| `Wechselrichter - maximale AC-Leistung` | Wird für Diagnose und Watt-Anzeige aus dem Solarman-Prozentwert genutzt. Die Solarman-Steuerung selbst schreibt Prozentwerte. |
| `Wechselrichter - fester Drosselwert` | Prozentwert, auf den `*_active_power_regulation` gesetzt wird, wenn Drosselung nötig ist. Default: 50 %. |
| `Wechselrichter wieder entdrosseln unter SoC` | Fällt mindestens ein Speicher unter diesen SoC oder wird Netzbezug erkannt, wird der Wechselrichter direkt wieder auf 100 % freigegeben. |

---

## 4. Berechnung der Referenz-Netzleistung

Die gemessene Netzleistung wird um bereits gesetzte Speicherbefehle bereinigt. Dadurch erkennt die Automation, was ohne aktuelle Speicherbefehle ungefähr passieren würde.

Formel:

```text
uncontrolled_grid_power_w = grid_power_w - current_input_total_w + current_output_total_w
```

Daraus entstehen:

```text
uncontrolled_import_w = max(0, uncontrolled_grid_power_w)
uncontrolled_surplus_w = max(0, -uncontrolled_grid_power_w)
```

Diese Korrektur ist wichtig, weil ein laufender Speicher-zu-Speicher-Transfer sonst fälschlich als normaler Hausverbrauch oder normale Einspeisung interpretiert werden könnte.

Beispiel bei aktivem Transfer:

```text
abgebender Speicher Output = 800 W
annehmender Speicher Input = 600 W
Netto-Beitrag zur Hausversorgung = 800 W - 600 W = 200 W
```

Der Blueprint betrachtet in diesem Fall nur 200 W als echte Hausversorgung. Dadurch wird ein bewusst gestarteter Transfer im nächsten Sekundenzyklus nicht sofort wieder beendet oder durch Wechselrichterdrosselung unterdrückt.

---

## 5. Betriebszustände und Priorität

Der Blueprint unterscheidet folgende Betriebszustände:

1. `emergency` - Notladung
2. `balance` - Speicher-zu-Speicher-Ausgleich über das Hausnetz
3. `surplus` - normale Überschussladung
4. `import` - Entladung zur Reduzierung von Netzbezug
5. `idle` - keine relevante Aktion

Die Priorität ist bewusst konservativ:

```text
Notladung > Speicher-zu-Speicher-Ausgleich > Überschussladung > Entladung gegen Netzbezug > Leerlauf
```

Notladung hat immer Vorrang. Während einer Notladung wird kein normaler Balance-Transfer gestartet.

---

## 6. Sicherheitslogik

### Fehlende Entitäten

Wenn Pflicht-Entitäten nicht automatisch aus den ausgewählten Geräten erkannt werden, wird eine persistente Home-Assistant-Benachrichtigung erzeugt und der Regelzyklus beendet.

Typische Ursachen:

- Es wurde nicht das physische Zendure-Speichergerät ausgewählt.
- Speicher 1 und Speicher 2 wurden vertauscht oder doppelt ausgewählt.
- Entity-IDs wurden manuell so umbenannt, dass typische Suffixe nicht mehr vorhanden sind.
- Die Solarman-Entität `*_active_power_regulation` fehlt.
- Das Off-Grid-Gerät ist nicht Speicher 1 oder Speicher 2.

### Ungültige Sensorwerte

Wenn kritische Sensoren `unknown`, `unavailable` oder ungültig sind, setzt der Blueprint beide Speicher sicher auf:

```text
input_limit = 0 W
output_limit = 0 W
```

Danach wird der aktuelle Regelzyklus beendet. Im nächsten Sekundenzyklus wird erneut geprüft.

### Wechselrichter nicht erreichbar

Wenn die Solarman-Entität des Wechselrichters nicht erreichbar ist, wird nur der Off-Grid-Port desjenigen Speichers ausgeschaltet, der in der Blueprint-Maske als Speicher mit angeschlossenem Wechselrichter ausgewählt wurde.

- Select-Entität: Option `off`
- Switch-Entität: `switch.turn_off`

Sobald der Wechselrichter wieder erreichbar ist, wird genau dieser Port wieder aktiviert:

- Select-Entität: Option `normal`
- Switch-Entität: `switch.turn_on`

Der Off-Grid-Port des anderen Speichers wird nicht verändert.

---

## 7. Normale Überschussladung

Normale Überschussladung wird aktiv, wenn:

```text
uncontrolled_surplus_w > Totband
```

und mindestens ein Speicher noch laden darf.

Ein Speicher darf laden, wenn sein SoC unterhalb der Vollgrenze liegt:

```text
storage_soc < full_soc - 0.5
```

Die gesamte Ladeleistung wird begrenzt durch:

```text
min(
  uncontrolled_surplus_w,
  max_house_grid_power_w,
  Summe der technischen input_limit-Grenzen der ladbaren Speicher
)
```

Die Verteilung auf beide Speicher erfolgt gewichtet nach:

```text
Ladegewicht = Gesamtkapazität in Wh * fehlende Prozentpunkte bis full_soc
```

Damit erhält ein größerer oder deutlich leererer Speicher mehr Überschussladung.

---

## 8. Normale Entladung zur Hausversorgung

Normale Entladung wird aktiv, wenn:

```text
uncontrolled_import_w > Totband
```

und mindestens ein Speicher entladen darf.

Ein Speicher darf nur entladen, wenn:

```text
SoC > reserve_soc
und
available_kwh > 0.01
```

Wenn ein Speicher diese Grenze erreicht, wird sein Ziel-Output hart auf 0 W gesetzt. Der andere Speicher übernimmt dann die Hausversorgung allein, soweit seine Leistungsgrenze aus `*_inverse_max_power` ausreicht.

### Entladegrenze je Speicher

Die Entladegrenze kommt automatisch aus:

```text
*_inverse_max_power
```

Falls das `max`-Attribut der Zendure-`output_limit`-Number kleiner ist, wird der kleinere Wert verwendet:

```text
storage_max_discharge_w = min(inverse_max_power, output_limit.max)
```

### Verteilung bei beiden vollen Speichern

Wenn beide Speicher voll sind und beide entladen dürfen, wird die Entladung im gewichteten Bereich gleichmäßig verteilt:

```text
Speicher 1 = 50 %
Speicher 2 = 50 %
```

Beispiel bei beiden Speichern mit 800 W Entladegrenze:

```text
Hausbedarf 1200 W
gewichteter Bereich bis 800 W: 400 W + 400 W
Zusatzbedarf 400 W: 200 W + 200 W
Ergebnis: 600 W + 600 W
```

Bei 1600 W Bedarf können beide Speicher jeweils bis zu 800 W liefern, sofern beide oberhalb der Entlade-Reserve liegen.

### Verteilung bei nicht vollen Speichern

Wenn nicht beide Speicher voll sind, wird bis zur größten einzelnen Entladegrenze eine gewichtete Verteilung verwendet. Dabei werden zwei Größen kombiniert:

1. verfügbare Energie aus `*_available_kwh`
2. aktuelle Ladeleistung aus Off-Grid und Solar

Die Ladeleistung wird so gebildet:

```text
charge_for_distribution = max(
  output_pack_power,
  abs(grid_off_power) + max(0, solar_input_power)
)
```

Dadurch wird ein Speicher auch dann korrekt bewertet, wenn er nur über Off-Grid, nur über Solar oder gleichzeitig über beide Quellen geladen wird.

Vereinfacht wird für die Entladeverteilung gerechnet:

```text
available_share = storage_available_wh / available_wh_sum
charge_share = charge_for_distribution / charge_for_distribution_sum
availability_factor = 0.5 + available_kwh / total_kwh

Entladegewicht = (available_share + 2 * charge_share) * availability_factor
```

Die Ladeleistung wird bewusst stärker gewichtet als die reine verfügbare Energie. Dadurch wird der Speicher, der gerade deutlich mehr Energie bekommt, stärker zur Hausversorgung herangezogen. Hat ein Speicher mehr verfügbare Energie und zugleich die höhere Ladeleistung, steigt sein Anteil entsprechend deutlich.

### Dominanzfall bei überproportional hoher Ladeleistung

Wenn ein Speicher eine deutlich überproportionale Ladeleistung hat, soll er die Hausversorgung allein übernehmen, statt nur anteilig stärker beteiligt zu werden. Der Blueprint bewertet eine Ladeleistung als dominant, wenn sie mindestens doppelt so hoch ist wie die Ladeleistung des anderen Speichers und zugleich die Hausversorgung bis zur eigenen `*_inverse_max_power`-Grenze abdecken kann.

Dann gilt:

```text
Speicher mit dominanter Ladeleistung liefert allein
bis maximal eigene inverse_max_power
```

Übersteigt der Hausbedarf diese Grenze, wird nur der Restbedarf auf den anderen Speicher verteilt.

Beispiel:

```text
Hausbedarf = 1200 W
Speicher 1 charge_for_distribution = 1500 W
Speicher 1 inverse_max_power = 800 W
Speicher 2 charge_for_distribution = 200 W

Speicher 1 übernimmt 800 W
Speicher 2 hilft mit 400 W aus
```

Wenn der Bedarf oberhalb der größten einzelnen Speichergrenze liegt, wird der zusätzliche Bedarf auf die noch freien Entladekapazitäten beider Speicher verteilt. Dadurch können beide Speicher parallel bis zu ihren jeweiligen `*_inverse_max_power`-Grenzen liefern.

---

## 9. Speicher-zu-Speicher-Ausgleich über das Hausnetz

Der Speicher-zu-Speicher-Ausgleich dient dazu, Überschüsse nicht unnötig ins Netz zu drücken oder den Wechselrichter sofort zu drosseln, sondern den anderen Speicher zu laden.

Der Ausgleich ist nur erlaubt, wenn alle Grundbedingungen erfüllt sind:

```text
keine aktive Notladung
abgebender Speicher darf entladen
annehmender Speicher darf laden
annehmender Speicher hat niedrigeren SoC
SoC-Vorsprung des abgebenden Speichers >= balance_soc_margin
Transferleistung >= balance_transfer_min_w
```

Die Richtung ist immer eindeutig:

- `1_to_2`, wenn Speicher 1 höher liegt und Speicher 2 niedriger liegt
- `2_to_1`, wenn Speicher 2 höher liegt und Speicher 1 niedriger liegt
- `none`, wenn keine Richtung zulässig ist

### Regel 1: Nicht voller Speicher mit deutlichem Quell-Überschuss

Bei nicht vollem abgebendem Speicher wird nur der Überschuss oberhalb des Parameters `surplus_exchange_threshold_w` übertragen.

Zuerst wird berechnet, wie viel Leistung für die Hausversorgung reserviert werden muss:

```text
current_net_storage_output_to_house_w = max(0, current_output_total_w - current_input_total_w)

balance_required_house_output_w = max(
  current_net_storage_output_to_house_w,
  uncontrolled_import_w
)
```

Dann wird je möglichem abgebendem Speicher berechnet:

```text
source_surplus_before_threshold = max(
  0,
  aktuelle Batterieladeleistung des abgebenden Speichers - balance_required_house_output_w
)
```

Zusätzlich muss dieser Quell-Überschuss größer sein als die aktuelle Einspeisung des anderen Speichers. Damit wird verhindert, dass ein Speicher nur deshalb transferiert, weil der andere bereits zur Hausversorgung einspeist.

Danach wird die konfigurierbare Toleranz abgezogen:

```text
source_surplus_above_threshold = max(
  0,
  source_surplus_before_threshold - surplus_exchange_threshold_w
)
```

Nur dieser Rest darf als Speicher-zu-Speicher-Transfer genutzt werden.

#### Beispiel für Regel 1

Gegeben:

```text
Hausverbrauch / reservierte Hausversorgung = 200 W
Speicher 1 lädt mit 300 W und speist aktuell 100 W ein
Speicher 2 lädt mit 1000 W und speist aktuell 100 W ein
surplus_exchange_threshold_w = 200 W
Speicher 1 hat niedrigeren SoC als Speicher 2
```

Berechnung für Speicher 2 als abgebender Speicher:

```text
source_surplus_before_threshold = 1000 W - 200 W = 800 W
source_surplus_above_threshold = 800 W - 200 W = 600 W
```

Wenn die technischen Limits es erlauben, lautet das Ziel:

```text
Speicher 2 output_limit = 200 W Hausversorgung + 600 W Transfer = 800 W
Speicher 1 input_limit = 600 W
```

Der Transfer wird also nicht als sofortiger Grund zur Wechselrichterdrosselung bewertet, weil die 600 W bewusst in Speicher 1 aufgenommen werden.

### Regel 2: Abgebender Speicher ist voll

Wenn genau ein Speicher voll ist und der andere Speicher noch laden kann, wird zuerst geprüft, ob der volle Speicher mehr aktuelle Ladeleistung hat, als für die Hausversorgung reserviert werden muss. Die Ladeleistung wird dabei wie oben aus `output_pack_power`, `*_grid_off_power` und `*_solar_input_power` gebildet.

Wenn diese Bedingung erfüllt ist, übernimmt der volle Speicher die komplette Hausversorgung bis zu seiner `*_inverse_max_power`-Grenze. Erst der darüber verbleibende Überschuss darf den anderen Speicher über das Hausnetz laden.

Vereinfacht:

```text
source_surplus_before_threshold = max(
  0,
  charge_for_distribution_full_storage - reserved_house_output
)

full_surplus_available = max(
  source_surplus_before_threshold,
  realer berechneter Netzüberschuss
) - zero_tolerance_w
```

Für volle Speicher wird nicht `surplus_exchange_threshold_w` abgezogen. Stattdessen gilt nur das Totband um die Nulleinspeisung. Der Transfer ist zusätzlich durch die freie Entladeleistung nach Hausversorgung und durch das input_limit des annehmenden Speichers begrenzt.

Beispiel:

```text
Speicher 1 ist voll
Speicher 2 ist nicht voll und hat niedrigeren SoC
reservierte Hausversorgung = 300 W
Speicher 1 charge_for_distribution = 1200 W
zero_tolerance_w = 40 W

Speicher 1 übernimmt zuerst 300 W Hausversorgung
verbleibender Vollspeicher-Überschuss = 1200 W - 300 W - 40 W = 860 W
Speicher 2 darf bis zu 860 W laden, begrenzt durch sein input_limit und die freie output-Leistung von Speicher 1
```

Dadurch wird verhindert, dass der volle Speicher weiter unnötig Überschuss erzeugt, während der zweite Speicher noch Energie aufnehmen könnte.

### Finale Transferbegrenzung

Der endgültige Transfer ist immer begrenzt durch:

```text
min(
  gewünschter Transfer,
  freie Entlade-/Transferleistung des abgebenden Speichers nach Hausversorgung,
  Ladegrenze des annehmenden Speichers,
  verfügbarer Quell-Überschuss
)
```

Es gibt kein separates Speicher-zu-Speicher-Transferlimit mehr. Die Grenze ist die Entlade-/Transferleistung des abgebenden Speichers aus `*_inverse_max_power`, abzüglich der für die Hausversorgung reservierten Leistung.

---

## 10. Notladung

Notladung wird aktiv, wenn mindestens ein Speicher unter den Notlade-Startwert fällt:

```text
SoC <= grid_charge_soc
```

Eine bereits laufende Notladung wird fortgesetzt, bis der Stop-Wert erreicht ist:

```text
SoC < grid_charge_stop_soc
und
Speicher ist bereits im AC-Modus input
und
aktuelles input_limit > min_change_w
```

### Netz-Notladung

Wenn kein anderer Speicher helfen darf, wird aus dem Hausnetz geladen. Die Gesamtleistung wird begrenzt durch:

```text
min(
  Anzahl niedriger Speicher * emergency_charge_w,
  max_house_grid_power_w,
  Summe der Ladegrenzen der niedrigen Speicher
)
```

Die Verteilung erfolgt nach:

```text
Notladegewicht = Gesamtkapazität in Wh * fehlende Prozentpunkte bis grid_charge_stop_soc
```

Ein größerer oder niedrigerer Speicher bekommt dadurch mehr Notladeleistung.

### Gegenseitige Notladeunterstützung

Ein Speicher darf den anderen bei Notladung nur versorgen, wenn:

```text
der annehmende Speicher Notladung braucht
abgebender Speicher keine Notladung braucht
abgebender Speicher entladen darf
abgebender Speicher SoC >= grid_charge_soc + mutual_emergency_soc_margin
abgebender Speicher hat höheren SoC als der annehmende Speicher
```

Wenn diese Bedingungen nicht erfüllt sind, erfolgt die Notladung aus dem Hausnetz.

Die gegenseitige Notladeleistung ist begrenzt durch:

```text
min(
  emergency_charge_w,
  Entladegrenze des abgebenden Speichers,
  Ladegrenze des annehmenden Speichers
)
```

---

## 11. Wechselrichter-Drosselung über Solarman

Der Wechselrichter wird über die Solarman-Entität `*_active_power_regulation` gesteuert. Diese Entität erwartet einen Prozentwert. Ab v17 berechnet der Blueprint die Drosselhöhe nicht mehr aus Hausverbrauch, aktuellem Restüberschuss oder einem Rampenschritt. Stattdessen wird bei Drosselbedarf direkt ein konfigurierbarer fester Prozentwert geschrieben.

### Drosselbedingung

Gedrosselt wird nur, wenn nach geplanter Speicheraufnahme noch Überschuss übrig bleibt und beide Speicher voll sind oder beide Speicher nicht mehr laden können. Dabei wird weiterhin berücksichtigt, ob Überschuss bewusst in einen Speicher fließen soll:

```text
surplus_for_inverter_throttle_w = max(
  0,
  uncontrolled_surplus_w - planned_surplus_absorption_w
)
```

Wenn ein Speicher-zu-Speicher-Transfer geplant ist, zählt die geplante Aufnahme des annehmenden Speichers als bewusste Überschussverwertung. Der Wechselrichter wird deshalb nicht gedrosselt, solange der andere Speicher den Überschuss aufnehmen soll.

### Drosselhöhe

Ist die Drosselbedingung erfüllt, wird der Solarman-Parameter direkt auf den konfigurierten festen Prozentwert gesetzt:

```text
inverter_target_percent = inverter_throttle_percent
```

Default für `inverter_throttle_percent` ist 50 %. Bei einem Wechselrichter mit 2000 W Maximalleistung entspricht das rechnerisch 1000 W, geschrieben wird aber der Prozentwert `50`.

### Entdrosseln

Der Wechselrichter wird direkt wieder auf 100 % freigegeben, wenn mindestens eine der Bedingungen erfüllt ist:

```text
Speicher 1 SoC < inverter_unthrottle_soc
oder
Speicher 2 SoC < inverter_unthrottle_soc
oder
uncontrolled_import_w > zero_tolerance_w
```

Wenn Solarman nicht erreichbar ist, wird kein neuer Prozentwert geschrieben. Stattdessen greift die Off-Grid-Sicherheitslogik.

---

## 12. Zielwerte je Speicher

Am Ende des Regelzyklus entstehen vier zentrale Zielwerte:

```text
storage1_target_input_w
storage1_target_output_w
storage2_target_input_w
storage2_target_output_w
```

Die Priorität bei der Zielwertbildung lautet:

1. Notladung oder gegenseitige Notladeunterstützung
2. Speicher-zu-Speicher-Balance
3. normale Überschussladung
4. normale Entladung gegen Netzbezug
5. sonst 0 W

Wenn ein Speicher eine positive Ladeleistung bekommt, wird sein AC-Modus auf `input` gesetzt. Andernfalls wird bei aktiver Entladung `output` gesetzt.

Werte werden nur geschrieben, wenn die Differenz zum aktuellen Wert mindestens `min_change_w` beträgt. Dadurch entstehen weniger unnötige Schreibvorgänge an Zendure.

---

## 13. Typische Zustände und Reaktion

### Zustand A: Beide Speicher sind voll

- Beide Speicher dürfen entladen.
- Normale Hausversorgung wird gleichmäßig verteilt.
- Wenn weiterhin Überschuss entsteht und ein Speicher doch noch annehmen kann, wird dieser Überschuss verwendet.
- Wenn kein Speicher mehr laden kann, wird der Wechselrichter über Solarman direkt auf den konfigurierten festen Drosselwert gesetzt.

### Zustand B: Speicher 1 ist leer bzw. an der unteren Reserve, Speicher 2 hat genug Energie

- Speicher 1 bekommt `output_limit = 0 W`.
- Speicher 2 übernimmt die Hausversorgung bis zu seiner `*_inverse_max_power`-Grenze.
- Wenn Speicher 1 zusätzlich unter dem Notlade-Startwert liegt, wird Notladung aktiv.
- Speicher 2 darf Speicher 1 nur unterstützen, wenn Speicher 2 oberhalb `grid_charge_soc + mutual_emergency_soc_margin` liegt und höheren SoC hat. Sonst wird aus dem Netz geladen.

### Zustand C: Speicher 2 hat sehr hohe Ladeleistung, Speicher 1 ist niedriger

- Der Blueprint prüft, ob Speicher 2 die aktuelle Hausversorgung voll tragen kann.
- Danach wird geprüft, ob die Batterieladeleistung von Speicher 2 oberhalb der Toleranz `surplus_exchange_threshold_w` liegt.
- Nur der Rest oberhalb dieser Toleranz wird an Speicher 1 übertragen.
- Speicher 1 muss niedrigeren SoC haben und laden können.

### Zustand D: Ein Speicher ist voll, der andere nicht

- Der volle Speicher kann den Überschuss an den niedrigeren Speicher übertragen.
- Dabei wird `surplus_exchange_threshold_w` nicht abgezogen.
- Es gelten trotzdem SoC-Richtung, Ladefähigkeit, Entladefähigkeit, technische Grenzen und Mindesttransferleistung.

### Zustand E: Es gibt nur kleinen Überschuss innerhalb des Totbands

- Keine neue Lade- oder Drosselaktion.
- Kleine Schwankungen werden ignoriert.
- Bestehende Werte werden nur geändert, wenn die Differenz mindestens `min_change_w` erreicht.

---

## 14. Hinweise zur Inbetriebnahme

1. Zuerst prüfen, ob alle automatisch erkannten Entitäten in Home Assistant vorhanden sind.
2. Shelly-Vorzeichen prüfen: Bei Netzbezug muss der intern verwendete Wert positiv sein.
3. `inverter_max_w` passend zum Wechselrichter setzen, damit Solarman-Prozentwerte korrekt berechnet werden.
4. `full_soc`, `reserve_soc`, `grid_charge_soc` und `grid_charge_stop_soc` an die gewünschte Akkuschonung anpassen.
5. `surplus_exchange_threshold_w` vorsichtig einstellen:
   - höherer Wert = weniger Speicher-zu-Speicher-Transfer
   - niedrigerer Wert = früherer Transfer bei Quell-Überschuss
6. `balance_soc_margin` kann auf 1 bis 3 Prozent erhöht werden, wenn bei fast gleichen SoC-Werten zu oft die Richtung wechseln würde.
7. Nicht parallel andere Automationen auf dieselben Zendure-`input_limit`, `output_limit` oder `ac_mode`-Entitäten schreiben lassen.

---

## 15. Kurzform der wichtigsten Formeln

```text
Netzreferenz ohne aktuelle Speicherbefehle:
uncontrolled_grid_power = grid_power - current_input_total + current_output_total

Netzbezug:
uncontrolled_import = max(0, uncontrolled_grid_power)

Überschuss:
uncontrolled_surplus = max(0, -uncontrolled_grid_power)

Netto-Speicherbeitrag zur Hausversorgung:
current_net_storage_output_to_house = max(0, current_output_total - current_input_total)

Für Balance reservierte Hausversorgung:
balance_required_house_output = max(current_net_storage_output_to_house, uncontrolled_import)

Quell-Überschuss bei nicht vollem Speicher:
source_surplus_before_threshold = max(0, battery_charge_power_source - balance_required_house_output)

Transferfähiger Überschuss bei nicht vollem Speicher:
source_surplus_above_threshold = max(0, source_surplus_before_threshold - surplus_exchange_threshold_w)

Finaler Speicher-zu-Speicher-Transfer:
balance_transfer = min(
  desired_transfer,
  source_max_discharge - reserved_house_output,
  target_max_charge,
  source_surplus_cap
)

Wechselrichter-Drosselwert:
active_power_regulation = inverter_throttle_percent, wenn Drosselung nötig ist
active_power_regulation = 100, wenn Entdrosselung nötig ist
```
