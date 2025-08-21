# mssql-cheat-sheet


## GUI

### DBeaver


### Microsoft Studio
- https://learn.microsoft.com/en-us/ssms/install/install





<br><br>

---

<br><br>


# Dump

## Restore


### Option #1
- https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver17&tabs=ssms#restore-a-backup
```
1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance.
2. Right-click the Databases node in Object Explorer and select Restore Database....
3. Select Device:, and then select the ellipses (...) to locate your backup file.
4. Select Add and navigate to where your .bak file is located. Select the .bak file and then select OK.
5. Select OK to close the Select backup devices dialog box.
6. Select OK to restore the backup of your database.
```






<br><br>
<br><br>

## Export

### SSMS

#### Option #1
- https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver17&tabs=ssms#take-a-backup
```
1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance.
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









<br><br>

---

<br><br>

## Docker

### Docker-compose

#### 2022
```yaml
# https://hub.docker.com/r/microsoft/mssql-server#environment-variables
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-dev
    
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
      MSSQL_SA_PASSWORD: "Test1234!"
      
    ports:
      - "1433:1433"
    
    volumes:
      # CRITICAL: Map only /data subdirectory - full /var/opt/mssql mapping crashes on Windows/WSL2
      # https://stackoverflow.com/questions/65307784/mssql-container-fails-to-start-when-mapping-volumes-after-upgrading-to-wsl2
      - ${MSSQL_HOME:-./data}:/var/opt/mssql/data
    
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P Test1234! -No -Q 'SELECT 1' || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

```
