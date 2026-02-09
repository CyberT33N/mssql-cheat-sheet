

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















