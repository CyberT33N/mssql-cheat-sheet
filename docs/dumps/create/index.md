


<br><br>
<br><br>

## Export




### Terminal




<details><summary>Click to expand..</summary>


### Variante A (empfohlen): Kompletter DB‑Backup‑Dump als `.bak` per Terminal

#### 1) SQL‑Instanz herausfinden
In PowerShell (als Admin hilfreich geht aber auch ohne):

```powershell
Get-Service | Where-Object { $_.Name -like "MSSQL*" -or $_.Name -like "SQL*" } | Select-Object Name, Status, DisplayName
```

Typische Servernamen für `sqlcmd -S`:
- Default Instance: `localhost` oder `.`
- Named Instance: `localhost\INSTANZNAME` (z. B. `.\SQLEXPRESS`)

#### 2) Prüfen, ob `sqlcmd` da ist
```powershell
Get-Command Invoke-Sqlcmd -ErrorAction SilentlyContinue
```


Falls vorhanden, Verbindung testen (Windows-Auth):
```powershell
Invoke-Sqlcmd -ServerInstance ".\Z1" -Database "master" -Query "SELECT @@SERVERNAME AS ServerName; SELECT name FROM sys.databases ORDER BY name;"
```


#### 3) Backup erstellen
```
Invoke-Sqlcmd -ServerInstance ".\Z1" -Database master -QueryTimeout 0 -Query "SET NOCOUNT ON; DECLARE @dir nvarchar(4000)=CAST(SERVERPROPERTY('InstanceDefaultBackupPath') AS nvarchar(4000)); IF (@dir IS NULL OR LTRIM(RTRIM(@dir))='') EXEC master..xp_instance_regread N'HKEY_LOCAL_MACHINE',N'Software\Microsoft\MSSQLServer\MSSQLServer',N'BackupDirectory',@dir OUTPUT; IF (@dir IS NULL OR LTRIM(RTRIM(@dir))='') THROW 50000,'Could not determine SQL Server backup directory.',1; IF RIGHT(@dir,1) NOT IN ('\','/') SET @dir+=N'\'; DECLARE @stamp nvarchar(32)=CONVERT(nvarchar(8),GETDATE(),112)+N'_'+REPLACE(CONVERT(nvarchar(12),GETDATE(),114),':',''); DECLARE @with nvarchar(200)=CASE WHEN CAST(SERVERPROPERTY('EngineEdition') AS int)=4 THEN N'WITH COPY_ONLY, INIT, CHECKSUM, STATS=10' ELSE N'WITH COPY_ONLY, COMPRESSION, INIT, CHECKSUM, STATS=10' END; DECLARE @db sysname,@file nvarchar(4000),@sql nvarchar(max); DECLARE cur CURSOR LOCAL FAST_FORWARD FOR SELECT name,@dir+REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(name,N'\',N'_'),N'/',N'_'),N':',N'_'),N'*',N'_'),N'?',N'_'),N'""',N'_'),N'<',N'_'),N'>',N'_'),N'|',N'_')+N'_FULL_'+@stamp+N'.bak' FROM sys.databases WHERE name<>N'tempdb' AND state_desc=N'ONLINE' ORDER BY name; OPEN cur; FETCH NEXT FROM cur INTO @db,@file; WHILE @@FETCH_STATUS=0 BEGIN SET @sql=N'BACKUP DATABASE '+QUOTENAME(@db)+N' TO DISK = N'''+REPLACE(@file,'''','''''')+N''' '+@with+N';'; EXEC sp_executesql @sql; PRINT @db+N' => '+@file; FETCH NEXT FROM cur INTO @db,@file; END CLOSE cur; DEALLOCATE cur; SELECT CAST(SERVERPROPERTY('ServerName') AS nvarchar(256)) AS ServerInstance;"
```


Donwload zu finden unter z.b.
C:\Program Files\Microsoft SQL Server\MSSQL16.Z1\MSSQL\Backup


</details>























<br><br>
<br><br>







### CLI

#### SqlPackage (Not verified)




<details><summary>Click to expand..</summary>



### Variante B: „Logischer Dump“ als `.bacpac` (wenn Restore auf anderer Umgebung geplant ist)
`SqlPackage.exe` ist oft mit SSMS / SQL Server Data Tools installiert.



SqlPackage finden:
```
Get-ChildItem -Path "C:\Program Files","C:\Program Files (x86)" -Recurse -ErrorAction SilentlyContinue -Filter SqlPackage.exe | Select-Object -ExpandProperty FullName
```




Beispiel:
```powershell
& "C:\Program Files (x86)\Microsoft SQL Server\120\DAC\bin\SqlPackage.exe" /Action:Export /SourceServerName:"ZA-APP-01\Z1" /SourceDatabaseName:"Z1_DB" /TargetFile:"C:\Backups\z1\Z1_DB.bacpac" /SourceTrustServerCertificate:True
```




</details>



































<br><br>
<br><br>


---





<br><br>
<br><br>



### SSMS






<br><br>
<br><br>


#### sql files

<details><summary>Click to expand..</summary>

### SSMS: Datenbank als einzelne SQL-Dateien exportieren (Schema **und** Daten)

- **SSMS öffnen** → Verbindung zur Instanz → **Datenbank `Z1`** auswählen
- Rechtsklick auf **`Z1`** → **Tasks** → **Generate Scripts…**
- **Choose Objects**: “Select specific database objects” → **Tables** auswählen (oder “Script entire database”)
- **Set Scripting Options**: als Output **File** wählen (ggf. “one file per object”)
- Jetzt kommt die entscheidende Stelle:
  - Klicke auf **Advanced / Erweitert…**
  - Suche die Option **„Datentypen, für die ein Skript erstellt wird.“**
  - Wenn du **beides** willst (Tabellen/Schema **und** Daten/VALUES), dann **MUSST** du hier **„Schema und Daten“** auswählen
    - Alternativ: **„Nur Daten“** (nur INSERT/VALUES) oder **„Nur Schema“** (nur CREATE/INDEX)
- **OK** → **Next** → **Finish**


</details>




<br><br>
<br><br>


#### Bak file

<details><summary>Click to expand..</summary>

#### Option #1
- https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver17&tabs=ssms#take-a-backup
```
1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance. Ansicht > Objekt Explorer
2. Expand the Databases node in Object Explorer.
3. Right-click the database, hover over Tasks, and select Back up....
4. Under Destination, confirm that the path for your backup is correct. If you need to change the path, select Remove to remove the existing path, and then Add to type in a new path. You can use the ellipses to navigate to a specific file.
5. Select OK to take a backup of your database.
```

#### Option #2
Connect > Right Click Connection > New Query
```
BACKUP DATABASE [Z1]
TO DISK = N'C:\SQLBackups\Z1_FULL_20250821.bak'
WITH INIT, CHECKSUM, STATS = 5;
```
- In my case Z1 is my database

Then press execute



</details>















