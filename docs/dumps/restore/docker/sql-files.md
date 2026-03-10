
### Docker (falls MSSQL in Container läuft)

Für den lokalen Dev-Container `mssql-dev` mit `sa` / `Test1234!` und der Datenbank `Z1`.

Wenn die SQL-Datei bestehende Objekte wie Tabellen erneut anlegt, muss die Datenbank vorher gelöscht und neu erstellt werden. Sonst kommen Fehler wie `There is already an object named ... in the database`.

```powershell
# Datei in den Container kopieren
docker cp "C:\git\test\test-mono\apps\test\data\dumps\pvs\z1\base\sql\without-timer\all-in-one\z1_base_2026-03-06_15_50_02.sql" mssql-dev:/tmp/z1_base_2026-03-06_15_50_02.sql

# Bestehende Verbindungen beenden, Datenbank Z1 löschen und neu anlegen
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d master -No -C -Q "IF DB_ID(N'Z1') IS NOT NULL BEGIN ALTER DATABASE [Z1] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [Z1]; END; CREATE DATABASE [Z1];"

# SQL-Datei in die neu erstellte Datenbank Z1 importieren
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d Z1 -No -C -t 0 `
    -i /tmp/z1_base_2026-03-06_15_50_02.sql
```

Falls du es als Einzeiler ausführen willst:

```powershell
docker cp "C:\git\test\test-mono\apps\test\data\dumps\pvs\z1\base\sql\without-timer\all-in-one\z1_base_2026-03-06_15_50_02.sql" mssql-dev:/tmp/z1_base_2026-03-06_15_50_02.sql; docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Test1234!" -d master -No -C -Q "IF DB_ID(N'Z1') IS NOT NULL BEGIN ALTER DATABASE [Z1] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [Z1]; END; CREATE DATABASE [Z1];"; docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Test1234!" -d Z1 -No -C -t 0 -i /tmp/z1_base_2026-03-06_15_50_02.sql
```