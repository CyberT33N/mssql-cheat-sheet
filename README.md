# mssql-cheat-sheet


## GUI

### DBeaver


### Microsoft Studio
- https://learn.microsoft.com/en-us/ssms/install/install





<br><br>

---

<br><br>


# Backup

## Microsoft Studio
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
