


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












