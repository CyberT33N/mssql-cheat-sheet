# Restore Dumps

## SQL Server Management Studio (SSMS)
- docs\dumps\restore\SQL Server Management Studio (SSMS)\index.md


### Option #1
- https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver17&tabs=ssms#restore-a-backup

1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance. Ansicht > Objekt Explorer

2. Right-click the Databases node in Object Explorer and select Restore Database....

3. Select Device:, and then select the ellipses (...) to locate your backup file.
  - e.g. /var/opt/mssql/data

  ## Kontext — Server-/Container-Perspektive bei Docker Compose mit MS SQL

⚠️ **WICHTIG** zu verstehen: Wenn DU z. B. mit **Docker Compose** arbeitest und **MS SQL** dort eingerichtet hast und DU hier den **Server** auswählst, befindest DU dich zu diesem Zeitpunkt auf dem **Server**.

📌 Das heißt: DU kannst nicht von deiner **Hostmaschine** aus einfach eine **Backup-Datei** aus dem **Downloads-Ordner** nehmen.

## Anweisung — Backup-Datei in den Container kopieren


➡️ Das heißt: DU **MUSST** die **Backup-Datei** zuerst in den **Container** kopieren.
```shell
docker cp "C:\Users\denni\Downloads\z1_base_2026-02-12_15_31_06.bak" "mssql-dev:/var/opt/mssql/data/z1_base_2026-02-12_15_31_06.bak"
```

4. Select Add and navigate to where your .bak file is located. Select the .bak file and then select OK.
  - Wenn man jetzt hier zum Beispiel mit Docker Compose einen **MSQL-Server** erstellt hat und dann einen lokalen Dateipfad gemoggt hat, hat man nur Zugriff darauf und dort wird man die **Backup-Datei** importieren müssen, damit man dann ab hier Zugriff darauf hat.

5. Select OK to close the Select backup devices dialog box.
6. Select OK to restore the backup of your database.










<br><br>
<br><br>

--- 


<br><br>
<br><br>


### Scripts


#### Powershell



#### Restore from bak file

docs\dumps\restore\scripts\powershell\bak-file\index.md

<br><br>




#### Restore from sql files
- docs\dumps\restore\scripts\powershell\sql\index.md
