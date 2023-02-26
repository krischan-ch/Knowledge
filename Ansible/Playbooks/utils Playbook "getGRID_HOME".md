
# Abstract

utils: getGRID_HOME ermittelt das aktuelle GRID_HOME auf Basis des im zentralen Oracle Inventory.

# Ablauf

- Inventory Pfad ermitteln (/etc/oraInst.loc)
- Inventory.xml File prüfen
- GRID_HOME Pfad via Suchbegriff 'CRS="true"' ermitteln
- Fehlerbehandlung wenn kein oder mehr als ein Eintrag gefunden wird
- GRID_HOME Pfad prüfen (olsnodes binary)

Stat-Modul prüft Datei **/etc/oraInst.loc** auf Existenz und Ansible lookup liest den Wert für Key **inventory_loc**.

# Inventory File prüfen

- Ansible Stat-Modul prüft Datei auf Vorhandensein
- GRID_HOME in inventory.xml suchen
- grep 'CRS="true"' /app/oraInventory/ContentsXML/inventory.xml | awk -F\" '{print $4}'
- getGRID_HOME liest den Wert des Keys "inventory_loc" aus der Datei /etc/oraInst.loc
- sucht anschliessend im File <inventory_loc>/ContentsXML/inventory.xml mittels grep nach 'CRS="true"'
- selektiert mittels awk und Delemiter '"' nach dem vierten Wert

# Tags:

[[Ansible]] - [[Playbook]] - [[ZKB]]