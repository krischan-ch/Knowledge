
# Kurzübersicht Archivierungs-/ILM-Optionen

> ILM = Information Lifecycle Management

## temporal validity support (Dokumentation)

Diese Option bietet die Möglichkeit, für eine Tabelle eine oder mehrere Gültigkeitsdimensionen zu definieren.

Damit wird definiert, welcher Datensatz zu welcher Zeit gültig ist/war und somit bei Abfragen entsprechend berücksichtig oder ignoriert werden muss.

Beispiel für zeitbasierende Gültigkeit wäre das Einstellungs- und Entlassungsdatum von Mitarbeitern in einer HR-Applikation.

> Lizenz: Advanced Compression Option (in ZKB vorhanden)

## flashback data archive / aka "total recall" (Dokumentation)

Die Option flashback data archive zeichnet sämtliche Änderungen an einer Tabelle auf und speichert diese in einem Tablespace ab. Da die Änderungen via Flashback Funktionalität gespeichert werden, wird hier vergleichsweise wenig Speicherplatz belegt.

Einige Formen von Table-DDL und einige Datentypen sind bei Verwendung von flashback data archive nicht mehr möglich.

> Lizenz: Advanced Compression Option (in ZKB vorhanden)

## Information Lifecycle Management (Summary)

Zusammen mit dem HeatMap Feature, welches auf der DB aktiviert wird, bietet diese Option die Möglichkeit policy-basierende, automatische Maintenance-Jobs für Tabellen zu erstellen.

So können z. Bsp. nicht oder nur wenig verwendete Daten automatisch komprimiert und/oder auf ein Low-Cost-Tablespace verschoben werden.

> Lizenz: Advanced Compression Option (in ZKB vorhanden)

# Tags:

[[HowTo]] - [[ILM]]

