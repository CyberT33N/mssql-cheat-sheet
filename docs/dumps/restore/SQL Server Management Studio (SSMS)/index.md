## SQL Server Management Studio (SSMS)


### Option #1
- https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver17&tabs=ssms#restore-a-backup
```
1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance. Ansicht > Objekt Explorer
2. Right-click the Databases node in Object Explorer and select Restore Database....
3. Select Device:, and then select the ellipses (...) to locate your backup file.
4. Select Add and navigate to where your .bak file is located. Select the .bak file and then select OK.
  - Wenn man jetzt hier zum Beispiel mit Docker Compose einen **MSQL-Server** erstellt hat und dann einen lokalen Dateipfad gemoggt hat, hat man nur Zugriff darauf und dort wird man die **Backup-Datei** importieren müssen, damit man dann ab hier Zugriff darauf hat.
5. Select OK to close the Select backup devices dialog box.
6. Select OK to restore the backup of your database.
```



