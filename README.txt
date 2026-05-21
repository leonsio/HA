Zendure/Solarman Nulleinspeisung Device Blueprint v12
======================================================

Installation
------------
Kopiere den Ordner "blueprints" in dein Home-Assistant-config-Verzeichnis oder importiere die YAML-Datei als Blueprint.

Aenderungen in v12
------------------
- Kommentare in allen variables:-, choose:- und conditions:-Bloecken wurden deutlich erweitert.
- Jeder Block beschreibt jetzt genauer, welche Eingangswerte verwendet werden, welche Entscheidung getroffen wird und welche Zielwerte daraus entstehen.
- Die Kommentare erklaeren insbesondere Sicherheitsstopps, Off-Grid-Umschaltung, Speicher-zu-Speicher-Transfer, kapazitaets-/ladeleistungsbasierte Entladeverteilung und Solarman-Prozentbegrenzung.
- Keine funktionale Aenderung an der Regelungslogik gegenueber v11.

Aenderungen in v11
------------------
- Es gibt jetzt zwei getrennte Eingabefelder fuer die maximale Entladeleistung:
  - Speicher 1 - maximale Entladeleistung, Default 800 W.
  - Speicher 2 - maximale Entladeleistung, Default 800 W.
- Diese Werte begrenzen die Entladung jedes Speichers separat. Falls die Zendure-Output-Number ein kleineres max-Attribut meldet, wird der kleinere Wert verwendet.
- Die bisherige kapazitaets- und ladeleistungsbasierte Verteilung gilt nur bis zur groessten wirksamen Einzel-Speicher-Entladegrenze.
  Beispiel bei 800 W / 800 W: Bis 800 W wird wie bisher nach SoC, *_available_kwh, *_total_kwh und Batterieladeleistung verteilt.
- Wird mehr als diese Einzel-Speicher-Grenze benoetigt, wird der Mehrbedarf auf die noch freien Entladekapazitaeten verteilt. Dadurch koennen beide Speicher gleichzeitig bis zu ihrer jeweiligen Grenze liefern, z. B. 2 x 800 W.
- Das Speicher-zu-Speicher-Transferlimit wurde entfernt. Die Transfergrenze ist jetzt die maximale Entladeleistung des abgebenden Speichers plus die Ladegrenze des annehmenden Speichers und der tatsaechlich verfuegbare Ueberschuss.
- Die normale Entladung zur Hausversorgung wird nicht mehr durch "Maximale Ladeleistung ueber das Hausnetz" begrenzt, sondern durch die Summe der entladefaehigen Speichergrenzen.
- Notlade-Transfer zwischen den Speichern nutzt ebenfalls die maximale Entladeleistung des abgebenden Speichers als Grenze.

Wesentliche Funktionen aus v10
------------------------------
- Wenn beide Speicher voll sind und beide entladen duerfen, wird die Einspeisung bis zur Einzel-Speicher-Grenze gleichmaessig 50/50 verteilt.
- Wenn beide Speicher nicht voll sind, wird die Einspeisung bis zur Einzel-Speicher-Grenze nach *_available_kwh im Verhaeltnis zu *_total_kwh, zusaetzlich gewichtet durch die aktuelle Batterieladeleistung, verteilt.
- Ein Speicher mit groesserer verfuegbarer Energie und hoeherer aktueller Batterieladeleistung wird dadurch staerker fuer die Einspeisung genutzt.
- Speicher an der unteren Entladegrenze werden explizit fuer Einspeisung/Entladung gesperrt:
  - SoC <= Entlade-Reserve SoC, oder
  - *_available_kwh <= 0.01 kWh.
- Wenn nur ein Speicher noch Energie oberhalb der unteren Entladegrenze hat, uebernimmt dieser Speicher die gesamte benoetigte Einspeisung bzw. Entladung bis zu seinem eigenen Leistungsgrenzwert. Der leere bzw. niedrige Speicher erhaelt 0 W Output.

Wesentliche Funktionen aus v8/v9
--------------------------------
- Der Regelzyklus laeuft jede Sekunde: time_pattern seconds: "/1".
- Speicher 1 und Speicher 2 werden als Zendure-Geraete der Integration "zendure_ha" ausgewaehlt; benoetigte Entitaeten werden ueber device_entities() gesucht.
- Der Speicher mit Wechselrichter am Off-Grid-Port wird separat ausgewaehlt. Nur dieses Geraet wird bei Wechselrichter-Ausfall am Off-Grid-Port geschaltet.
- Der Solarman-/Deye-Wechselrichter wird ueber number.*_active_power_regulation in Prozent begrenzt.
- Speicher-zu-Speicher-Transfer erfolgt nur bei realem Ueberschuss:
  1. wenn der abgebende Speicher voll ist, oder
  2. wenn die berechnete Einspeisung den konfigurierbaren Grenzwert "Tolerierte Einspeisung vor Speicher-Transfer" ueberschreitet.
- Der annehmende Speicher muss einen niedrigeren SoC haben als der abgebende Speicher.
- Bei nicht vollem abgebendem Speicher wird nur dessen Ladeleistungs-Ueberschuss gegenueber dem anderen Speicher verwendet.
- Jeder "variables:", "choose:" und "conditions:"-Block ist ausfuehrlich kommentiert.

Wichtige Parameter
------------------
- Speicher 1 - maximale Entladeleistung: Grenze fuer Hausversorgung und Transfer, wenn Speicher 1 abgibt. Default 800 W.
- Speicher 2 - maximale Entladeleistung: Grenze fuer Hausversorgung und Transfer, wenn Speicher 2 abgibt. Default 800 W.
- Maximale Ladeleistung ueber das Hausnetz: Grenze fuer normale Ueberschussladung und Netz-Notladung. Sie begrenzt nicht mehr die normale Entladung und nicht mehr den Speicher-zu-Speicher-Transfer.
- Entlade-Reserve SoC: Unterer SoC-Wert. Speicher mit SoC <= diesem Wert werden nicht mehr fuer Einspeisung/Entladung verwendet.
- Speicher mit Wechselrichter am Off-Grid-Port: muss Speicher 1 oder Speicher 2 sein. Nur dieses Geraet wird fuer die Off-Grid-Sicherheitsabschaltung geschaltet.
- Tolerierte Einspeisung vor Speicher-Transfer: Standard 1000 W. Nicht volle Speicher tauschen erst den Ueberschuss oberhalb dieses Werts aus.
- Reserve fuer gegenseitige Notaufladung ueber SoC-Limit: Standard 20 Prozentpunkte. Nur darueber darf ein Speicher den anderen bei Notladung versorgen.
- Wechselrichter maximale AC-Leistung: Basis fuer die Umrechnung von Watt in Prozent fuer Solarman *_active_power_regulation.

Hinweis
-------
Der Blueprint verwendet ausschliesslich Zendure-Geraete aus der Integration "zendure_ha" fuer Speicher 1 und Speicher 2. Die benoetigten Entitaeten werden ueber device_entities() anhand typischer Zendure-HA-Suffixe gesucht, z. B. *_total_kwh, *_available_kwh, *_ac_mode, *_input_limit, *_output_limit und Off-Grid-Suffixe wie *_grid_off_mode, *_off_grid_mode oder *_offgrid_mode.
