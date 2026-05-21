Zendure/Solarman Nulleinspeisung Device Blueprint v8
======================================================

Installation
------------
Kopiere den Ordner "blueprints" in dein Home-Assistant-config-Verzeichnis oder importiere die YAML-Datei als Blueprint.

Aenderungen in v8
-----------------
- Neues Eingabefeld "Speicher mit Wechselrichter am Off-Grid-Port".
- Darueber wird festgelegt, ob der Wechselrichter am Off-Grid-Port von Speicher 1 oder Speicher 2 angeschlossen ist.
- Bei nicht erreichbarem Solarman-Wechselrichter wird nur noch der Off-Grid-Port dieses ausgewaehlten Speichers ausgeschaltet.
- Sobald der Wechselrichter wieder erreichbar ist, wird ebenfalls nur dieser ausgewaehlte Off-Grid-Port wieder eingeschaltet.
- Der Off-Grid-Port des jeweils anderen Speichers bleibt unveraendert.
- Wenn das ausgewaehlte Off-Grid-Geraet nicht Speicher 1 oder Speicher 2 ist oder dort keine passende Off-Grid-Entitaet gefunden wird, erzeugt der Blueprint eine persistente Benachrichtigung und bricht den Regelzyklus sicher ab.

Wesentliche Funktionen aus v5/v6/v7
-----------------------------------
- Der Regelzyklus laeuft jede Sekunde: time_pattern seconds: "/1".
- Speicher-zu-Speicher-Transfer erfolgt nur bei realem Ueberschuss:
  1. wenn der abgebende Speicher voll ist, oder
  2. wenn die berechnete Einspeisung den konfigurierbaren Grenzwert "Tolerierte Einspeisung vor Speicher-Transfer" ueberschreitet.
- Der Grenzwert hat den Default 1000 W. Alles oberhalb dieses Werts wird, begrenzt durch Speicherlimits und die maximale Hausnetz-Leistung, zum Speicher mit niedrigerem SoC uebertragen.
- Der annehmende Speicher muss weiterhin einen niedrigeren SoC haben als der abgebende Speicher.
- Bei nicht vollem abgebendem Speicher wird zusaetzlich nur dessen Ladeleistungs-Ueberschuss gegenueber dem anderen Speicher verwendet.
- Die Wechselrichterdrosselung beruecksichtigt geplante Speicheraufnahme. Wenn Ueberschuss bewusst zum Laden des anderen Speichers verwendet wird, wird der Wechselrichter nicht sofort gedrosselt.
- Jeder "variables:", "choose:" und "conditions:"-Block ist kommentiert.

Wichtige Parameter
------------------
- Speicher mit Wechselrichter am Off-Grid-Port: muss Speicher 1 oder Speicher 2 sein. Nur dieses Geraet wird fuer die Off-Grid-Sicherheitsabschaltung geschaltet.
- Maximale Leistung ueber das Hausnetz: absolute Leistungsgrenze fuer Speicherladung, Entladung und Speicher-zu-Speicher-Transfer.
- Tolerierte Einspeisung vor Speicher-Transfer: Standard 1000 W. Nicht volle Speicher tauschen erst den Ueberschuss oberhalb dieses Werts aus.
- Reserve fuer gegenseitige Notaufladung ueber SoC-Limit: Standard 20 Prozentpunkte. Nur darueber darf ein Speicher den anderen bei Notladung versorgen.
- Wechselrichter maximale AC-Leistung: Basis fuer die Umrechnung von Watt in Prozent fuer Solarman *_active_power_regulation.

Hinweis
-------
Der Blueprint verwendet ausschliesslich Zendure-Geraete aus der Integration "zendure_ha" fuer Speicher 1 und Speicher 2. Die benoetigten Entitaeten werden ueber device_entities() anhand typischer Zendure-HA-Suffixe gesucht, z. B. *_total_kwh, *_ac_mode, *_input_limit, *_output_limit und Off-Grid-Suffixe wie *_grid_off_mode, *_off_grid_mode oder *_offgrid_mode.
