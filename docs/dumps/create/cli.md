



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




