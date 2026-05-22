Zendure/Solarman Nulleinspeisung Device Blueprint v13
======================================================

Installation
------------
Kopiere den Ordner "blueprints" in dein Home-Assistant-config-Verzeichnis oder importiere die YAML-Datei als Blueprint.

Aenderungen in v13
------------------
- Die maximale Entlade-/Transferleistung wird nicht mehr pro Speicher in der Blueprint-Maske abgefragt.
- Stattdessen sucht der Blueprint je Zendure-Speicher automatisch die Entitaet *_inverse_max_power und verwendet deren aktuellen Wert als Grenze fuer:
  - normale Entladung zur Hausversorgung,
  - Speicher-zu-Speicher-Transfer,
  - gegenseitige Notladeunterstuetzung.
- Falls die Zendure-output_limit-Number ein kleineres max-Attribut meldet, wird weiterhin der kleinere sichere Wert verwendet.
- Die AC-Modus-Optionen werden nicht mehr abgefragt:
  - Laden setzt fest den Zendure-HA-Select-Wert "input".
  - Entladen setzt fest den Zendure-HA-Select-Wert "output".
- Die Off-Grid-Modus-Optionen werden nicht mehr abgefragt:
  - Off-Grid-Port abschalten setzt fest "off".
  - Off-Grid-Port einschalten setzt fest "normal".
- Die Pflichtentitaetspruefung wurde um *_inverse_max_power fuer Speicher 1 und Speicher 2 erweitert.
- Kommentare und Beschreibungen wurden auf die automatische Verwendung von *_inverse_max_power und feste Modusnamen angepasst.

Wesentliche Funktionen aus v12/v11
----------------------------------
- Jeder "variables:", "choose:" und "conditions:"-Block ist ausfuehrlich kommentiert.
- Der Regelzyklus laeuft jede Sekunde: time_pattern seconds: "/1".
- Speicher 1 und Speicher 2 werden als Zendure-Geraete der Integration "zendure_ha" ausgewaehlt; benoetigte Entitaeten werden ueber device_entities() gesucht.
- Der Speicher mit Wechselrichter am Off-Grid-Port wird separat ausgewaehlt. Nur dieses Geraet wird bei Wechselrichter-Ausfall am Off-Grid-Port geschaltet.
- Der Solarman-/Deye-Wechselrichter wird ueber number.*_active_power_regulation in Prozent begrenzt.
- Die kapazitaets- und ladeleistungsbasierte Entladeverteilung gilt bis zur groessten aktuell verfuegbaren Einzel-Speicher-Grenze aus *_inverse_max_power.
- Wird mehr Leistung benoetigt, wird der Mehrbedarf auf die noch freien Entladekapazitaeten verteilt. Dadurch koennen beide Speicher gleichzeitig bis zu ihrem jeweiligen *_inverse_max_power-Wert liefern.
- Speicher-zu-Speicher-Transfer nutzt kein separates Transferlimit. Die Transfergrenze ist der *_inverse_max_power-Wert des abgebenden Speichers plus die Ladegrenze des annehmenden Speichers und der tatsaechlich verfuegbare Ueberschuss.
- Die normale Entladung zur Hausversorgung wird nicht durch "Maximale Ladeleistung ueber das Hausnetz" begrenzt, sondern durch die Summe der entladefaehigen Speichergrenzen aus *_inverse_max_power.

Wesentliche Speicherlogik
-------------------------
- Wenn beide Speicher voll sind und beide entladen duerfen, wird die Einspeisung bis zur Einzel-Speicher-Grenze gleichmaessig 50/50 verteilt.
- Wenn beide Speicher nicht voll sind, wird die Einspeisung bis zur Einzel-Speicher-Grenze nach *_available_kwh im Verhaeltnis zu *_total_kwh, zusaetzlich gewichtet durch die aktuelle Batterieladeleistung, verteilt.
- Ein Speicher mit groesserer verfuegbarer Energie und hoeherer aktueller Batterieladeleistung wird dadurch staerker fuer die Einspeisung genutzt.
- Speicher an der unteren Entladegrenze werden explizit fuer Einspeisung/Entladung gesperrt:
  - SoC <= Entlade-Reserve SoC, oder
  - *_available_kwh <= 0.01 kWh.
- Wenn nur ein Speicher noch Energie oberhalb der unteren Entladegrenze hat, uebernimmt dieser Speicher die gesamte benoetigte Einspeisung bzw. Entladung bis zu seinem aus *_inverse_max_power gelesenen Leistungsgrenzwert. Der leere bzw. niedrige Speicher erhaelt 0 W Output.

Wichtige Parameter
------------------
- Maximale Ladeleistung ueber das Hausnetz: Grenze fuer normale Ueberschussladung und Netz-Notladung. Sie begrenzt nicht die normale Entladung und nicht den Speicher-zu-Speicher-Transfer.
- Entlade-Reserve SoC: Unterer SoC-Wert. Speicher mit SoC <= diesem Wert werden nicht mehr fuer Einspeisung/Entladung verwendet.
- Speicher mit Wechselrichter am Off-Grid-Port: muss Speicher 1 oder Speicher 2 sein. Nur dieses Geraet wird fuer die Off-Grid-Sicherheitsabschaltung geschaltet.
- Tolerierte Einspeisung vor Speicher-Transfer: Standard 1000 W. Nicht volle Speicher tauschen erst den Ueberschuss oberhalb dieses Werts aus.
- Reserve fuer gegenseitige Notaufladung ueber SoC-Limit: Standard 20 Prozentpunkte. Nur darueber darf ein Speicher den anderen bei Notladung versorgen.
- Wechselrichter maximale AC-Leistung: Basis fuer die Umrechnung von Watt in Prozent fuer Solarman *_active_power_regulation.

Hinweis
-------
Der Blueprint verwendet ausschliesslich Zendure-Geraete aus der Integration "zendure_ha" fuer Speicher 1 und Speicher 2. Die benoetigten Entitaeten werden ueber device_entities() anhand typischer Zendure-HA-Suffixe gesucht, z. B. *_total_kwh, *_available_kwh, *_ac_mode, *_input_limit, *_output_limit, *_inverse_max_power und Off-Grid-Suffixe wie *_grid_off_mode, *_off_grid_mode oder *_offgrid_mode.
