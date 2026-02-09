



### CLI

#### SqlPackage




<details><summary>Click to expand..</summary>



### Variante B: „Logischer Dump“ als `.bacpac` (wenn Restore auf anderer Umgebung geplant ist)
`SqlPackage.exe` ist oft mit SSMS / SQL Server Data Tools installiert.

#### SQL‑Instanz herausfinden
In PowerShell (als Admin hilfreich geht aber auch ohne):

```powershell
Get-Service | Where-Object { $_.Name -like "MSSQL*" -or $_.Name -like "SQL*" } | Select-Object Name, Status, DisplayName
```


---

**DB-Namen prüfen (Einzeiler):**

```powershell
Invoke-Sqlcmd -ServerInstance ".\Z1" -Database master -Query "SELECT name FROM sys.databases ORDER BY name;"
```

**Zugriff auf genau `Z1_DB` prüfen (Einzeiler):**

```powershell
Invoke-Sqlcmd -ServerInstance ".\Z1" -Database master -Query "SELECT HAS_DBACCESS('Z1_DB') AS HasAccess;"
```

- `HasAccess = 1` → Name stimmt + Zugriff ok
- `HasAccess = 0` → kein Zugriff (oder Name stimmt nicht)

---



SqlPackage finden:
```
Get-ChildItem -Path "C:\Program Files","C:\Program Files (x86)" -Recurse -ErrorAction SilentlyContinue -Filter SqlPackage.exe | Select-Object -ExpandProperty FullName
```

Falls es nicht existiert installieren:
- https://go.microsoft.com/fwlink/?linkid=2338524


Beispiel:
- Erstellt Ordner `C:\Backups``und macht dann Backup
```powershell
New-Item -ItemType Directory -Force -Path "C:\Backups\z1" | Out-Null; & "C:\Program Files\Microsoft SQL Server\170\DAC\bin\SqlPackage.exe" /Action:Export /SourceServerName:".\Z1" /SourceDatabaseName:"Z1" /TargetFile:"C:\Backups\z1\Z1.bacpac" /SourceTrustServerCertificate:True
```




</details>




