# Download
- https://learn.microsoft.com/de-de/sql/tools/sqlcmd/sqlcmd-download-install?view=sql-server-ver17&tabs=windows
  - E.g. https://github.com/microsoft/go-sqlcmd/releases/download/v1.9.0/sqlcmd-amd64.msi 


# Install

```shell
winget install sqlcmd
```


In manchen Fällen gibt es Probleme, zum Beispiel bei „About:*“ im neuen Terminalfenster.

Falls es global nicht verfügbar ist, kann man es hiermit eintragen.
```shell
$env:PATH += ";C:\Program Files\sqlcmd"
```





### sqlcmd über Kommandozeile (empfohlen für große Dateien)

Die zuverlässigste Methode für große SQL-Dateien.

```cmd
sqlcmd -S localhost -d DatabaseName -U Username -P Password -i "C:\path\to\dump.sql"
```

**Parameter:**
| Parameter | Beschreibung |
|-----------|--------------|
| `-S` | Server-Name (z.B. `localhost`, `localhost\SQLEXPRESS`) |
| `-d` | Ziel-Datenbank |
| `-U` | Benutzername |
| `-P` | Passwort |
| `-i` | Pfad zur SQL-Datei |
| `-f 65001` | UTF-8 Encoding (falls nötig) |

**Beispiel mit Windows Authentication:**
```cmd
sqlcmd -S localhost -d MyDatabase -E -i "C:\dumps\large_dump.sql"
```

**Beispiel mit SQL Authentication:**
```cmd
sqlcmd -S localhost -d MyDatabase -U sa -P "mypassword" -i "C:\dumps\large_dump.sql"
```



### Weitere Ressourcen

- [Microsoft Docs: sqlcmd Utility](https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility)