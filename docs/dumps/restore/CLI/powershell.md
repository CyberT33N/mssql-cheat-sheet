


### PowerShell mit Invoke-Sqlcmd

Für automatisierte Workflows oder Skripte.

```powershell
Invoke-Sqlcmd -ServerInstance "localhost" `
              -Database "MyDatabase" `
              -Username "sa" `
              -Password "mypassword" `
              -InputFile "C:\dumps\large_dump.sql"
```

**Mit Windows Authentication:**
```powershell
Invoke-Sqlcmd -ServerInstance "localhost" `
              -Database "MyDatabase" `
              -InputFile "C:\dumps\large_dump.sql"
```




---

### Weitere Ressourcen


- [Microsoft Docs: Invoke-Sqlcmd](https://learn.microsoft.com/en-us/powershell/module/sqlserver/invoke-sqlcmd)
