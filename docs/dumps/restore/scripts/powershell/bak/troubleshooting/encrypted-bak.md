
## Troubleshooting-Dokument — Import eines verschlüsselten `.bak`

### 1) Symptom
Beim Import/Analyse des Backups (oft schon bei `RESTORE FILELISTONLY` oder `RESTORE HEADERONLY`) kommt ein Fehler ähnlich:

- “Cannot find server certificate with thumbprint `0x…`”
- “RESTORE … is terminating abnormally”

### 2) Wichtiges Verständnis: Warum kann das passieren, obwohl dein Backup kein `WITH ENCRYPTION` nutzt?
Es gibt zwei Hauptursachen für “verschlüsseltes Backup”:

#### A) **Backup Encryption** (explizit)
- Das Backup wurde mit `BACKUP DATABASE ... WITH ENCRYPTION (SERVER CERTIFICATE = ...)` erstellt.
- Dann ist klar: Ohne Zertifikat+Private Key auf dem Zielserver kein Restore.

#### B) **TDE (Transparent Data Encryption)** (implizit)
- Die Datenbank ist TDE-verschlüsselt.
- Dann sind **Backups automatisch verschlüsselt**, selbst wenn dein `BACKUP DATABASE` keinerlei `WITH ENCRYPTION` enthält.
- Effekt: Restore auf einem anderen SQL Server erfordert das **TDE-Zertifikat inkl. Private Key** vom Quellserver.

Das erklärt das scheinbare Paradox: *“Unser Code setzt kein `WITH ENCRYPTION`, aber das `.bak` ist trotzdem verschlüsselt.”*

### 3) Schnellprüfung auf dem Quell-SQL-Server (SSMS)
Führe auf dem Quellserver (dort wo das Backup erzeugt wurde) aus:

#### 3.1 TDE-Status der DB prüfen
```sql
SELECT
  name,
  is_encrypted
FROM sys.databases
WHERE name = 'Z1';

SELECT
  DB_NAME(database_id) AS database_name,
  encryption_state,
  encryptor_type,
  key_algorithm,
  key_length
FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID('Z1');
```

- `is_encrypted = 1` und ein Eintrag in `sys.dm_database_encryption_keys` → sehr starkes Indiz für **TDE**.

#### 3.2 Backup-Metadaten prüfen (wenn msdb Historie da ist)
```sql
SELECT TOP (20)
  database_name,
  backup_finish_date,
  is_encrypted,
  encryptor_thumbprint
FROM msdb.dbo.backupset
WHERE database_name = 'Z1'
ORDER BY backup_finish_date DESC;
```

- `is_encrypted = 1` und `encryptor_thumbprint` gesetzt → Backup ist verschlüsselt (egal ob durch TDE oder explizite Backup Encryption).

### 4) Was ist für Restore/Import notwendig?
Wenn das Backup verschlüsselt ist, brauchst du auf dem **Ziel-SQL-Server**:

- **Zertifikat (.cer)**
- **Private Key (.pvk)**
- ggf. **Passwort** für den Private Key-Export

Danach muss das Zertifikat in `master` importiert werden, **bevor** `RESTORE` funktioniert.

### 5) Wie kommt man an Zertifikat + Private Key?
Das kommt **vom Quell-SQL-Server** (oder IT/DBA):

```sql
BACKUP CERTIFICATE [<CertificateName>]
TO FILE = 'C:\secure\backup-encryption.cer'
WITH PRIVATE KEY (
  FILE = 'C:\secure\backup-encryption.pvk',
  ENCRYPTION BY PASSWORD = '<StrongPassword>'
);
```

### 6) Import auf dem Ziel-SQL-Server (einmalig)
```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<SomeStrongPassword>';
ALTER MASTER KEY ADD ENCRYPTION BY SERVICE MASTER KEY;

CREATE CERTIFICATE [<CertificateNameOrNewName>]
FROM FILE = 'C:\secure\backup-encryption.cer'
WITH PRIVATE KEY (
  FILE = 'C:\secure\backup-encryption.pvk',
  DECRYPTION BY PASSWORD = '<StrongPassword>'
);
```

### 7) Danach: Restore normal durchführen
- `RESTORE FILELISTONLY ...`
- `RESTORE DATABASE ... WITH MOVE ...`

### 8) Operative Empfehlung (wichtig)
- **Nicht**: `.pvk` im normalen Export-Artefakt mitschicken (“export + delete”) – das ist ein Secret-Handling-Risiko.
- **Stattdessen**: separater **Provisioning-Prozess**:
  - IT/DBA importiert das Zertifikat einmalig in den Ziel-SQL-Server, oder
  - ein Admin-Runbook/Skript macht das kontrolliert (RBAC/Audit).

### 9) Wenn du in SSMS ein Backup “ohne Encryption” machen willst
- Ein **neues** Backup ohne Backup-Encryption ist in SSMS normalerweise möglich.
- Aber wenn die DB **TDE** nutzt, bleibt das Backup trotzdem verschlüsselt (implizit) → dann hilft nur Zertifikats-Provisionierung oder ein nicht-TDE Quellbackup (falls zulässig).
