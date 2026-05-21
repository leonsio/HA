Generischer Home Assistant Blueprint: Nulleinspeisung mit zwei Speichern und Solarman-Wechselrichter
===============================================================================================

Datei im ZIP:
  blueprints/automation/local/generic_storage_inverter_nulleinspeisung_blueprint.yaml

Installation:
  1. Den Ordner "blueprints" aus diesem ZIP in das Home-Assistant-Konfigurationsverzeichnis kopieren.
  2. In Home Assistant zu Einstellungen -> Automationen & Szenen -> Blueprints wechseln.
  3. Blueprint neu laden und daraus eine Automation erstellen.
  4. Die Entitäten für Speicher 1, Speicher 2, Netzleistung und Wechselrichter auswählen.

Wichtige Eingaben:
  - Netzanschluss - Gesamtleistung:
      Sensor in Watt. Erwartet wird positiv = Netzbezug, negativ = Einspeisung.
      Falls dein Sensor andersherum zählt, "Netzleistung Vorzeichen invertieren" aktivieren.

  - Speicher 1 / Speicher 2 - Gesamtkapazität _total_kwh:
      Sensor mit float-Wert in kWh. Der Blueprint rechnet intern in Wh um.

  - Wechselrichter - Solarman _active_power_regulation:
      Number-Entität der Solarman-Integration. Der Blueprint setzt hier 0 bis 100 Prozent.
      Die Prozentvorgabe wird aus "Wechselrichter - maximale AC-Leistung" berechnet.
      Beispiel: 2000 W Maximalleistung und 1000 W Zielwert ergeben 50 Prozent.

  - Maximale Leistung über das Hausnetz:
      Default 1000 W. Dieser Wert begrenzt Lade-/Entlade- und Speicher-zu-Speicher-Leistung
      über das Hausnetz.

  - Mindest-SoC-Vorsprung des abgebenden Speichers:
      Default 0 %. Speicher-zu-Speicher-Ausgleich wird nur erlaubt, wenn der annehmende
      Speicher einen niedrigeren SoC hat als der abgebende Speicher. Bei z. B. 2 % muss
      der abgebende Speicher mindestens 2 Prozentpunkte höher liegen.

Ausgleichslogik:
  - Der Speicher mit deutlich höherer Batterieladeleistung kann Leistung über das Hausnetz
    an den schwächer geladenen Speicher übertragen.
  - Das passiert nicht nur, wenn ein Speicher voll ist, sondern auch bei asymmetrischer
    Ladeleistung.
  - Voraussetzung: Der annehmende Speicher muss einen niedrigeren SoC haben als der
    abgebende Speicher.
  - Zusätzlich wird der Transfer durch den gemessenen Ladeleistungsüberschuss des
    abgebenden Speichers begrenzt. Dadurch wird nicht planmäßig gespeicherte Energie
    verschoben, sondern nur der aktuelle Überschuss.
  - Beispiel:
      Speicher 2 lädt mit 2000 W, Speicher 1 mit 1000 W, Hausverbrauch 200 W.
      Wenn Speicher 1 einen niedrigeren SoC als Speicher 2 hat, bei Schwellwert 300 W
      und Transferfaktor 50 Prozent berechnet der Blueprint ca. 500 W Ausgleich.
      Speicher 2 gibt dann zusätzlich ab, Speicher 1 lädt zusätzlich. Der Hausverbrauch
      wird berücksichtigt, sodass die resultierende Vorgabe z. B. Speicher 2 Output
      ca. 700 W und Speicher 1 Input ca. 500 W sein kann.
      Quell- und Zielwerte werden durch "Maximale Leistung über das Hausnetz" gedeckelt.

Notaufladung:
  - Wenn ein Speicher unter "Netz-Nachladen starten unter SoC" fällt, wird Notaufladung aktiviert.
  - Ein anderer Speicher darf nur dann über das Hausnetz helfen, wenn sein SoC mindestens um
    "Reserve für gegenseitige Notaufladung über SoC-Limit" über dem Startlimit liegt und
    höher ist als der SoC des annehmenden Speichers.
  - Andernfalls wird aus dem Hausnetz geladen.

Hinweise:
  - Die AC-Modus-Optionen müssen exakt zu deiner Integration passen.
    Häufige Werte sind "input"/"output", "INPUT"/"OUTPUT" oder "charge"/"discharge".
  - Nicht parallel mit einer anderen Automation betreiben, die dieselben Input-/Output-Limits
    oder den AC-Modus der Speicher setzt.
  - Die Regelung läuft alle 5 Sekunden.
