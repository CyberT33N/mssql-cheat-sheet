# Dumps

## Restore

### SQL

#### Without sql client

Warnung: Das folgende Skript läuft in-memory. Das heißt, die SQL-Datei wird eingelesen, und wir überspringen den Schritt über einen SQL-Client wie SQLCMD komplett.

Das funktioniert für kleine Dumps. Für größere Dumps mit zum Beispiel über 1 Gigabyte würde es allerdings nicht funktionieren.