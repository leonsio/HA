Zendure/Solarman Nulleinspeisung Device Blueprint v14
======================================================

Installation
------------
Kopiere den Ordner "blueprints" in dein Home-Assistant-config-Verzeichnis oder importiere die YAML-Datei als Blueprint.

Aenderungen in v14
------------------
- Die Speicher-zu-Speicher-Transferlogik wurde ueberarbeitet.
- Regel 1 fuer nicht volle Speicher:
  - Der abgebende Speicher muss zuerst die aktuell benoetigte Haus-Einspeisung voll abdecken koennen.
  - Berechnung der reservierten Haus-Einspeisung: max(Netto-Speichereinspeisung ins Haus, berechneter Netzbezug ohne Speicherbefehle).
  - Netto-Speichereinspeisung ins Haus = aktuelle Output-Limits beider Speicher minus aktuelle Input-Limits beider Speicher. Dadurch bleibt bei bereits laufendem Transfer nur die echte Hausversorgung uebrig.
  - Quell-Ueberschuss: aktuelle Batterieladeleistung des abgebenden Speichers minus reservierte Haus-Einspeisung.
  - Nur der Anteil oberhalb "surplus_exchange_threshold_w" wird an den anderen Speicher weitergegeben.
  - Beispiel: 1000 W Batterieladung, 100 W aktueller Output von Speicher 1, 100 W aktueller Output von Speicher 2 und 200 W Schwelle ergeben 1000 W - 200 W - 200 W = 600 W Transfer.
- Der abgebende Speicher uebernimmt im Balance-Modus die Hausversorgung plus den Speicher-zu-Speicher-Transfer.
  - Beispiel: 200 W Hausversorgung plus 600 W Transfer ergeben 800 W Output am abgebenden Speicher.
  - Der annehmende Speicher bekommt nur die Transferleistung als input_limit.
- Regel 2 fuer volle Speicher:
  - Wenn ein Speicher voll ist, wird sein verfuegbarer Ueberschuss an den zweiten Speicher uebergeben.
  - Dabei werden SoC-Richtung, Ladefaehigkeit des annehmenden Speichers, Entladefaehigkeit des abgebenden Speichers, *_inverse_max_power, input_limit-Maximum und Mindesttransfer weiterhin beruecksichtigt.
  - Fuer volle Speicher wird die konfigurierbare Transfer-Schwelle nicht abgezogen; es gilt nur das Totband der Nulleinspeisung.
- Transfer bleibt nur erlaubt, wenn der annehmende Speicher einen niedrigeren SoC hat als der abgebende Speicher.
- Die Wechselrichterdrosselung beruecksichtigt weiterhin geplante Speicheraufnahme, damit ein bewusst gestarteter Transfer nicht sofort durch Drosselung unterbunden wird.
- Blueprint-Name und interne Versionskennung wurden auf v14 aktualisiert.

Wesentliche Funktionen aus v13/v12/v11
--------------------------------------
- Jeder "variables:", "choose:" und "conditions:"-Block ist ausfuehrlich kommentiert.
- Der Regelzyklus laeuft jede Sekunde: time_pattern seconds: "/1".
- Speicher 1 und Speicher 2 werden als Zendure-Geraete der Integration "zendure_ha" ausgewaehlt; benoetigte Entitaeten werden ueber device_entities() gesucht.
- Die maximale Entlade-/Transferleistung wird nicht pro Speicher in der Blueprint-Maske abgefragt.
- Stattdessen sucht der Blueprint je Zendure-Speicher automatisch die Entitaet *_inverse_max_power und verwendet deren aktuellen Wert als Grenze fuer:
  - normale Entladung zur Hausversorgung,
  - Speicher-zu-Speicher-Transfer,
  - gegenseitige Notladeunterstuetzung.
- Falls die Zendure-output_limit-Number ein kleineres max-Attribut meldet, wird weiterhin der kleinere sichere Wert verwendet.
- Die AC-Modus-Optionen werden nicht abgefragt:
  - Laden setzt fest den Zendure-HA-Select-Wert "input".
  - Entladen setzt fest den Zendure-HA-Select-Wert "output".
- Die Off-Grid-Modus-Optionen werden nicht abgefragt:
  - Off-Grid-Port abschalten setzt fest "off".
  - Off-Grid-Port einschalten setzt fest "normal".
- Der Speicher mit Wechselrichter am Off-Grid-Port wird separat ausgewaehlt. Nur dieses Geraet wird bei Wechselrichter-Ausfall am Off-Grid-Port geschaltet.
- Der Solarman-/Deye-Wechselrichter wird ueber number.*_active_power_regulation in Prozent begrenzt.
- Die kapazitaets- und ladeleistungsbasierte Entladeverteilung gilt bis zur groessten aktuell verfuegbaren Einzel-Speicher-Grenze aus *_inverse_max_power.
- Wird mehr Leistung benoetigt, wird der Mehrbedarf auf die noch freien Entladekapazitaeten verteilt. Dadurch koennen beide Speicher gleichzeitig bis zu ihrem jeweiligen *_inverse_max_power-Wert liefern.
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
- Tolerierter Quell-Ueberschuss vor Speicher-Transfer: Standard 1000 W. Nicht volle Speicher tauschen nur den Anteil oberhalb dieses Werts aus, nachdem die aktuelle Haus-Einspeisung reserviert wurde.
- Mindestleistung fuer Speicher-Ausgleich: Unterhalb dieser Leistung wird kein zusaetzlicher Speicher-zu-Speicher-Ausgleich gestartet.
- Reserve fuer gegenseitige Notaufladung ueber SoC-Limit: Standard 20 Prozentpunkte. Nur darueber darf ein Speicher den anderen bei Notladung versorgen.
- Wechselrichter maximale AC-Leistung: Basis fuer die Umrechnung von Watt in Prozent fuer Solarman *_active_power_regulation.

Hinweis
-------
Der Blueprint verwendet ausschliesslich Zendure-Geraete aus der Integration "zendure_ha" fuer Speicher 1 und Speicher 2. Die benoetigten Entitaeten werden ueber device_entities() anhand typischer Zendure-HA-Suffixe gesucht, z. B. *_total_kwh, *_available_kwh, *_ac_mode, *_input_limit, *_output_limit, *_inverse_max_power und Off-Grid-Suffixe wie *_grid_off_mode, *_off_grid_mode oder *_offgrid_mode.
