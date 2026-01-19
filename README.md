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
1. Launch SQL Server Management Studio (SSMS) and connect to your SQL Server instance. Ansicht > Objekt Explorer
2. Right-click the Databases node in Object Explorer and select Restore Database....
3. Select Device:, and then select the ellipses (...) to locate your backup file.
4. Select Add and navigate to where your .bak file is located. Select the .bak file and then select OK.
  - Wenn man jetzt hier zum Beispiel mit Docker Compose einen **MSQL-Server** erstellt hat und dann einen lokalen Dateipfad gemoggt hat, hat man nur Zugriff darauf und dort wird man die **Backup-Datei** importieren müssen, damit man dann ab hier Zugriff darauf hat.
5. Select OK to close the Select backup devices dialog box.
6. Select OK to restore the backup of your database.
```






### Scripts

#### Powershell


<details><summary>Click to expand..</summary>


This folder contains a PowerShell script to import the Z1 SQL dumps located at `data/dumps/pvs/z1` into a local MSSQL instance (Docker).


### Force (destructive, default: off)

`-Force` will **drop and re-create** the target database before importing. This is the easiest way to get a clean re-run.

```powershell
.\scripts\pvs\z1\databases\dumps\import-dumps.ps1 `
  -ConnectionString "Server=localhost,1433;Database=master;User Id=sa;Password=Test1234!;Encrypt=Optional;TrustServerCertificate=True;" `
  -Database timer `
  -Force:$true `
  -Confirm:$false `
  -MaxBatchChars 9000000 `
  -Verbose
```

### Notes

- The dumps contain references like `timer.dbo.*`, so `-Database timer` is the expected target.
- `Encrypt=Optional` (from the docs) is normalized to an ADO.NET compatible value during execution.
- **`-EnsureDatabase`**: creates the target database if it does not exist (non-destructive).
- **`-Force`**: drops and re-creates the target database before importing (destructive; default is off).
- **`-Confirm:$false`**: disables interactive confirmations for `-EnsureDatabase/-Force` and per-file import steps (useful for automation).
- **`-MaxBatchChars`**: splits very large SQL batches (e.g. a huge INSERT batch) into smaller chunks to reduce SQL Server memory pressure while parsing/executing.
- **`-ResolveDependencies`**: imports files in a dependency-safe order (prevents missing referenced table errors).
- **`-DeferCreateIndex`**: moves `CREATE INDEX` blocks to the end of each file to speed up large inserts.
- Re-runs: if you already imported once, use `-Force` (or use a fresh database name) to avoid "There is already an object named ...".


### Optional: store the connection string in an environment variable

```powershell
$env:Z1_MSSQL_CONNECTION_STRING = "Server=localhost,1433;Database=master;User Id=sa;Password=***;Encrypt=Optional;TrustServerCertificate=True;"
.\scripts\pvs\z1\databases\dumps\import-dumps.ps1 -Database timer -EnsureDatabase -Verbose
```




```powershell
<#
.SYNOPSIS
    Imports Z1 SQL dump files into a local MSSQL instance.
.DESCRIPTION
    Imports all *.sql files from the Z1 dump directory (default: data\dumps\pvs\z1)
    into a Microsoft SQL Server using an ADO.NET connection (no external modules required).

    Features:
    - Supports WhatIf/Confirm via ShouldProcess
    - Robust batch splitting on standalone GO lines
    - Optional per-file transaction
    - Configurable command timeout
    - Object output summary for automation

    Connection options:
    - Provide -ConnectionString directly (recommended for CI)
    - Or provide -Credential + -Server/-Port/-Database
    - Or set environment variable Z1_MSSQL_CONNECTION_STRING

    Defaults match the referenced development environment documentation:
    Server=localhost,1433;Database=master;User Id=sa;Password=<...>;Encrypt=Optional;TrustServerCertificate=True;
.PARAMETER SqlDirectory
    Directory containing the Z1 *.sql dump files to import.
    Default resolves to the repository path: data\dumps\pvs\z1
.PARAMETER ConnectionString
    ADO.NET connection string used to connect to MSSQL.
    If not provided, the script uses Z1_MSSQL_CONNECTION_STRING or builds one from -Server/-Port/-Database/-Credential.
.PARAMETER Server
    MSSQL host name or IP. Default is 'localhost'.
.PARAMETER Port
    MSSQL TCP port. Default is 1433.
.PARAMETER Database
    Initial database to connect to. Default is 'timer' (Z1 dumps reference objects like timer.dbo.*).
    Note: dump files may contain USE statements which will switch databases during execution.
.PARAMETER EnsureDatabase
    Ensures that the target database exists before importing (creates it if missing).
    Requires permissions to create databases (e.g. 'sa' in local dev).
.PARAMETER Force
    DANGEROUS (destructive): Drops and re-creates the target database before importing.
    Use this for clean re-runs in local development to avoid "There is already an object named ..." errors.
    Requires permissions to drop/create databases (e.g. 'sa' in local dev).
.PARAMETER Credential
    SQL authentication credential (e.g. 'sa'). If not provided, -ConnectionString (or env var) must include credentials.
.PARAMETER CommandTimeoutSec
    SQL command timeout for each batch, in seconds. Default is 600.
.PARAMETER MaxBatchChars
    Maximum size (characters) of a single SQL batch sent to SQL Server.
    Large dump files can contain very large batches (e.g. many INSERT statements without GO), which can trigger SQL Server memory pressure while parsing/compiling.
    Default is 500000.
.PARAMETER UseTransaction
    Wrap each file import in a transaction (BEGIN/COMMIT). Useful for smaller imports.
    If a file fails, the transaction is rolled back for that file.
.PARAMETER ContinueOnError
    Continue importing remaining files if a file fails. Failures are reported in the summary object.
.PARAMETER AutoIdentityInsert
    Automatically retries batches that fail with "Cannot insert explicit value for identity column..."
    by enabling IDENTITY_INSERT for the affected table for the duration of the batch.
    Default is enabled.
.PARAMETER ResolveDependencies
    When enabled, the script tries to import SQL files in a dependency-safe order by analyzing CREATE TABLE / REFERENCES statements.
    This prevents errors like: "Foreign key ... references invalid table 'person'" caused by importing a dependent table before its referenced table exists.
    Default is enabled.
.PARAMETER DeferCreateIndex
    When enabled, moves simple CREATE INDEX blocks (delimited by GO) to the end of each file before execution.
    This typically speeds up large imports because inserts don't have to maintain nonclustered indexes while loading data.
    Default is enabled.
.PARAMETER FileName
    Import only these specific SQL file names (e.g. 'patient.sql', 'person.sql'). Case-insensitive.
    When omitted, all *.sql files in SqlDirectory are imported.
.PARAMETER ExcludeFileName
    Exclude these specific SQL file names from import. Case-insensitive.
.EXAMPLE
    # Import all Z1 dumps using an explicit connection string
    .\import-dumps.ps1 -ConnectionString "Server=localhost,1433;Database=timer;User Id=sa;Password=Test1234!;Encrypt=False;TrustServerCertificate=True;" -EnsureDatabase -Verbose
.EXAMPLE
    # Import a subset, safe preview
    .\import-dumps.ps1 -FileName patient.sql,person.sql -WhatIf
.EXAMPLE
    # Use environment variable for secrets
    $env:Z1_MSSQL_CONNECTION_STRING = "Server=localhost,1433;Database=timer;User Id=sa;Password=***;Encrypt=False;TrustServerCertificate=True;"
    .\import-dumps.ps1 -Verbose
#>
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'High')]
param(
    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$SqlDirectory,

    [Parameter(Mandatory = $false)]
    [string]$ConnectionString,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$Server = 'localhost',

    [Parameter(Mandatory = $false)]
    [ValidateRange(1, 65535)]
    [int]$Port = 1433,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$Database = 'timer',

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [System.Management.Automation.PSCredential]$Credential,

    [Parameter(Mandatory = $false)]
    [ValidateRange(1, 86400)]
    [int]$CommandTimeoutSec = 600,

    [Parameter(Mandatory = $false)]
    [ValidateRange(10000, 20000000)]
    [int]$MaxBatchChars = 500000,

    [Parameter(Mandatory = $false)]
    [switch]$UseTransaction,

    [Parameter(Mandatory = $false)]
    [switch]$ContinueOnError,

    [Parameter(Mandatory = $false)]
    [switch]$EnsureDatabase,

    [Parameter(Mandatory = $false)]
    [switch]$Force,

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [bool]$AutoIdentityInsert = $true,

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [bool]$ResolveDependencies = $true,

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [bool]$DeferCreateIndex = $true,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string[]]$FileName,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string[]]$ExcludeFileName
)

begin {
    Set-StrictMode -Version 2.0

    function Get-RepoRoot {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$StartDirectory
        )

        $current = $StartDirectory
        for ($i = 0; $i -lt 10; $i++) {
            $packageJson = Join-Path -Path $current -ChildPath 'package.json'
            $gitDir = Join-Path -Path $current -ChildPath '.git'

            if (Test-Path -LiteralPath $packageJson -PathType Leaf) {
                return $current
            }

            if (Test-Path -LiteralPath $gitDir -PathType Container) {
                return $current
            }

            $parent = Split-Path -Path $current -Parent
            if ([string]::IsNullOrWhiteSpace($parent) -or ($parent -eq $current)) {
                break
            }
            $current = $parent
        }

        throw "Unable to determine repository root from '$StartDirectory'. Please pass -SqlDirectory explicitly."
    }

    function Get-TextFromFile {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$LiteralPath
        )

        if (-not (Test-Path -LiteralPath $LiteralPath -PathType Leaf)) {
            throw "File not found: '$LiteralPath'"
        }

        $reader = $null
        try {
            # UTF-8 fallback + BOM detection
            $reader = New-Object System.IO.StreamReader($LiteralPath, [System.Text.Encoding]::UTF8, $true)
            return $reader.ReadToEnd()
        }
        finally {
            if ($null -ne $reader) {
                $reader.Dispose()
            }
        }
    }

    function Get-TextPrefixFromFile {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$LiteralPath,

            [Parameter(Mandatory = $false)]
            [ValidateRange(1024, 2000000)]
            [int]$MaxChars = 200000
        )

        if (-not (Test-Path -LiteralPath $LiteralPath -PathType Leaf)) {
            throw "File not found: '$LiteralPath'"
        }

        $reader = $null
        try {
            $reader = New-Object System.IO.StreamReader($LiteralPath, [System.Text.Encoding]::UTF8, $true)
            $buffer = New-Object char[] $MaxChars
            $read = $reader.Read($buffer, 0, $MaxChars)
            if ($read -le 0) {
                return ''
            }
            return (New-Object string ($buffer, 0, $read))
        }
        finally {
            if ($null -ne $reader) {
                $reader.Dispose()
            }
        }
    }

    function Split-SqlBatches {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$SqlText
        )

        # Split on standalone "GO" (case-insensitive) lines.
        # This is the common sqlcmd/SSMS batch separator behavior.
        $pattern = '(?im)^\s*GO\s*(?:--.*)?$'
        $parts = [System.Text.RegularExpressions.Regex]::Split($SqlText, $pattern)

        foreach ($part in $parts) {
            $batch = $part.Trim()
            if (-not [string]::IsNullOrWhiteSpace($batch)) {
                $batch
            }
        }
    }

    function Split-LargeSqlBatch {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Batch,

            [Parameter(Mandatory = $true)]
            [ValidateRange(10000, 20000000)]
            [int]$MaxBatchChars
        )

        if ($Batch.Length -le $MaxBatchChars) {
            return ,$Batch
        }

        # Best-effort splitting on line boundaries to avoid sending huge command texts to SQL Server.
        # These Z1 dump files typically have one statement per line (especially INSERTs).
        $lines = $Batch -split "(\r?\n)"
        $sb = New-Object System.Text.StringBuilder

        foreach ($token in $lines) {
            # Keep original newlines (the split keeps delimiters as separate tokens).
            if (($sb.Length + $token.Length) -gt $MaxBatchChars -and $sb.Length -gt 0) {
                $out = $sb.ToString().Trim()
                if (-not [string]::IsNullOrWhiteSpace($out)) {
                    $out
                }
                [void]$sb.Clear()
            }

            [void]$sb.Append($token)
        }

        $final = $sb.ToString().Trim()
        if (-not [string]::IsNullOrWhiteSpace($final)) {
            $final
        }
    }

    function Get-LastIdentifierPart {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$MultipartName
        )

        $parts = @($MultipartName.Split('.') | ForEach-Object { $_.Trim() } | Where-Object { -not [string]::IsNullOrWhiteSpace($_) })
        $last = $parts[$parts.Count - 1]
        if ($last.StartsWith('[', [System.StringComparison]::Ordinal) -and $last.EndsWith(']', [System.StringComparison]::Ordinal)) {
            $last = $last.Substring(1, $last.Length - 2)
        }
        return $last
    }

    function Get-CreateTableNamesFromSqlText {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$SqlText
        )

        $names = New-Object 'System.Collections.Generic.HashSet[string]' ([System.StringComparer]::OrdinalIgnoreCase)
        $pattern = '(?im)^\s*create\s+table\s+([^\s(]+)'
        $matches = [System.Text.RegularExpressions.Regex]::Matches($SqlText, $pattern)
        foreach ($m in $matches) {
            $raw = $m.Groups[1].Value
            $tbl = Get-LastIdentifierPart -MultipartName $raw
            if (-not [string]::IsNullOrWhiteSpace($tbl)) {
                [void]$names.Add($tbl)
            }
        }
        return $names
    }

    function Get-ReferencedTableNamesFromSqlText {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$SqlText
        )

        $names = New-Object 'System.Collections.Generic.HashSet[string]' ([System.StringComparer]::OrdinalIgnoreCase)
        $pattern = '(?im)\breferences\s+([^\s,(;]+)'
        $matches = [System.Text.RegularExpressions.Regex]::Matches($SqlText, $pattern)
        foreach ($m in $matches) {
            $raw = $m.Groups[1].Value.Trim()
            $tbl = Get-LastIdentifierPart -MultipartName $raw
            if (-not [string]::IsNullOrWhiteSpace($tbl)) {
                [void]$names.Add($tbl)
            }
        }
        return $names
    }

    function Resolve-SqlFileImportOrder {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.IO.FileInfo[]]$SqlFiles
        )

        # Stable topo-sort by parsing a prefix of each file (tables + references).
        $fileByTable = @{}
        $tablesByFile = @{}
        $refsByFile = @{}

        foreach ($f in $SqlFiles) {
            $prefix = Get-TextPrefixFromFile -LiteralPath $f.FullName -MaxChars 200000
            $defined = Get-CreateTableNamesFromSqlText -SqlText $prefix
            $refs = Get-ReferencedTableNamesFromSqlText -SqlText $prefix
            $tablesByFile[$f.FullName] = $defined
            $refsByFile[$f.FullName] = $refs

            foreach ($t in $defined) {
                if (-not $fileByTable.ContainsKey($t)) {
                    $fileByTable[$t] = $f.FullName
                }
            }
        }

        # Build dependency graph: file -> set(files it depends on)
        $deps = @{}
        $inDegree = @{}
        foreach ($f in $SqlFiles) {
            $deps[$f.FullName] = New-Object 'System.Collections.Generic.HashSet[string]' ([System.StringComparer]::OrdinalIgnoreCase)
            $inDegree[$f.FullName] = 0
        }

        foreach ($f in $SqlFiles) {
            $fKey = $f.FullName
            foreach ($r in $refsByFile[$fKey]) {
                if ($fileByTable.ContainsKey($r)) {
                    $depFile = $fileByTable[$r]
                    if ($depFile -and ($depFile -ne $fKey)) {
                        if ($deps[$fKey].Add($depFile)) {
                            $inDegree[$fKey] = $inDegree[$fKey] + 1
                        }
                    }
                }
            }
        }

        # Kahn algorithm with stable ordering by original name sort
        $byName = @{}
        foreach ($f in $SqlFiles) { $byName[$f.FullName] = $f.Name }

        $comparison = [System.Comparison[string]]{
            param($a, $b)
            return [string]::Compare($byName[$a], $byName[$b], $true)
        }

        $queue = New-Object System.Collections.Generic.List[string]
        foreach ($k in $inDegree.Keys) {
            if ($inDegree[$k] -eq 0) { $queue.Add($k) | Out-Null }
        }
        $queue.Sort($comparison)

        $resultKeys = New-Object System.Collections.Generic.List[string]
        while ($queue.Count -gt 0) {
            $n = $queue[0]
            $queue.RemoveAt(0)
            $resultKeys.Add($n) | Out-Null

            foreach ($m in $deps.Keys) {
                if ($deps[$m].Contains($n)) {
                    $deps[$m].Remove($n) | Out-Null
                    $inDegree[$m] = $inDegree[$m] - 1
                    if ($inDegree[$m] -eq 0) {
                        $queue.Add($m) | Out-Null
                        $queue.Sort($comparison)
                    }
                }
            }
        }

        if ($resultKeys.Count -ne $SqlFiles.Count) {
            # Cycle or unresolved: fall back to name sort
            return @($SqlFiles | Sort-Object -Property Name)
        }

        $fileMap = @{}
        foreach ($f in $SqlFiles) { $fileMap[$f.FullName] = $f }
        return @($resultKeys | ForEach-Object { $fileMap[$_] })
    }

    function Defer-CreateIndexBlocks {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$SqlText
        )

        # This is a conservative transformation:
        # - split by GO
        # - move blocks that start with "CREATE INDEX" / "CREATE UNIQUE INDEX" to the end
        # Note: keeps relative order among moved index blocks.
        $blocks = @(Split-SqlBatches -SqlText $SqlText)
        if ($blocks.Count -le 1) { return $SqlText }

        $indexBlocks = New-Object System.Collections.Generic.List[string]
        $mainBlocks = New-Object System.Collections.Generic.List[string]

        foreach ($b in $blocks) {
            if ($b -match '(?im)^\s*create\s+(unique\s+)?index\b') {
                $indexBlocks.Add($b) | Out-Null
            }
            else {
                $mainBlocks.Add($b) | Out-Null
            }
        }

        if ($indexBlocks.Count -eq 0) { return $SqlText }

        # Reassemble with GO separators so downstream logic remains the same.
        $all = New-Object System.Collections.Generic.List[string]
        foreach ($b in $mainBlocks) { $all.Add($b) | Out-Null }
        foreach ($b in $indexBlocks) { $all.Add($b) | Out-Null }
        return (($all | ForEach-Object { $_.Trim() } | Where-Object { $_ }) -join "`r`nGO`r`n")
    }

    function New-MssqlConnectionString {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Server,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 65535)]
            [int]$Port,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Database,

            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Management.Automation.PSCredential]$Credential
        )

        $user = $Credential.UserName
        $pwd = $Credential.GetNetworkCredential().Password

        # Dev-friendly defaults for local Docker MSSQL: Encrypt=False and TrustServerCertificate=True.
        return "Server=$Server,$Port;Database=$Database;User Id=$user;Password=$pwd;Encrypt=False;TrustServerCertificate=True;"
    }

    function Normalize-ConnectionString {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$ConnectionString
        )

        # Some docs/tools use 'Encrypt=Optional'. System.Data.SqlClient expects a boolean.
        $normalized = [System.Text.RegularExpressions.Regex]::Replace(
            $ConnectionString,
            '(?i)(^|;)\s*Encrypt\s*=\s*Optional\s*(?=;|$)',
            '$1Encrypt=False'
        )

        return $normalized
    }

    function Set-ConnectionStringDatabase {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$ConnectionString,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Database
        )

        $withoutDb = [System.Text.RegularExpressions.Regex]::Replace(
            $ConnectionString,
            '(?i)(^|;)\s*(Database|Initial\s+Catalog)\s*=\s*[^;]*',
            ''
        )

        $trimmed = $withoutDb.Trim().TrimEnd(';')
        if ([string]::IsNullOrWhiteSpace($trimmed)) {
            return "Database=$Database;"
        }

        return "$trimmed;Database=$Database;"
    }

    function Open-MssqlConnection {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$ConnectionString
        )

        # Prefer System.Data.SqlClient for Windows PowerShell 5.1 compatibility
        $connection = New-Object System.Data.SqlClient.SqlConnection($ConnectionString)
        $connection.Open()
        return $connection
    }

    function Invoke-MssqlBatch {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Data.SqlClient.SqlConnection]$Connection,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Batch,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 86400)]
            [int]$CommandTimeoutSec,

            [Parameter(Mandatory = $false)]
            [System.Data.SqlClient.SqlTransaction]$Transaction
        )

        $command = $null
        try {
            $command = $Connection.CreateCommand()
            $command.CommandTimeout = $CommandTimeoutSec
            $command.CommandType = [System.Data.CommandType]::Text
            $command.CommandText = $Batch
            if ($null -ne $Transaction) {
                $command.Transaction = $Transaction
            }
            [void]$command.ExecuteNonQuery()
        }
        finally {
            if ($null -ne $command) {
                $command.Dispose()
            }
        }
    }

    function ConvertTo-BracketedSqlIdentifier {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Identifier
        )

        $trimmed = $Identifier.Trim()
        if ($trimmed.StartsWith('[', [System.StringComparison]::Ordinal) -and $trimmed.EndsWith(']', [System.StringComparison]::Ordinal)) {
            # Strip existing brackets to normalize escaping
            $trimmed = $trimmed.Substring(1, $trimmed.Length - 2)
        }

        $escaped = $trimmed.Replace(']', ']]')
        return "[$escaped]"
    }

    function ConvertTo-BracketedMultipartName {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Name
        )

        # Handles: table | schema.table | db.schema.table
        $parts = @($Name.Split('.') | ForEach-Object { $_.Trim() } | Where-Object { -not [string]::IsNullOrWhiteSpace($_) })
        if ($parts.Count -eq 0) {
            throw "Invalid SQL identifier: '$Name'"
        }

        $bracketedParts = foreach ($p in $parts) {
            ConvertTo-BracketedSqlIdentifier -Identifier $p
        }
        return ($bracketedParts -join '.')
    }

    function Get-LastIdentifierPart {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$MultipartName
        )

        $parts = @($MultipartName.Split('.') | ForEach-Object { $_.Trim() } | Where-Object { -not [string]::IsNullOrWhiteSpace($_) })
        $last = $parts[$parts.Count - 1]
        if ($last.StartsWith('[', [System.StringComparison]::Ordinal) -and $last.EndsWith(']', [System.StringComparison]::Ordinal)) {
            $last = $last.Substring(1, $last.Length - 2)
        }
        return $last
    }

    function Find-InsertTargetForTable {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Batch,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$TableName
        )

        # Match targets like: INSERT INTO timer.dbo.aufgaben3 ( ... )
        $pattern = '(?im)^\s*INSERT\s+INTO\s+([^\s(]+)'
        $matches = [System.Text.RegularExpressions.Regex]::Matches($Batch, $pattern)
        foreach ($m in $matches) {
            $target = $m.Groups[1].Value
            $last = Get-LastIdentifierPart -MultipartName $target
            if ($last.Equals($TableName, [System.StringComparison]::OrdinalIgnoreCase)) {
                return $target
            }
        }

        return $TableName
    }

    function Invoke-MssqlBatchWithIdentityInsertHandling {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Data.SqlClient.SqlConnection]$Connection,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Batch,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 86400)]
            [int]$CommandTimeoutSec,

            [Parameter(Mandatory = $false)]
            [System.Data.SqlClient.SqlTransaction]$Transaction,

            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [bool]$AutoIdentityInsert
        )

        $identityInsertTarget = $null
        try {
            $attempt = 0
            while ($true) {
                try {
                    Invoke-MssqlBatch -Connection $Connection -Batch $Batch -CommandTimeoutSec $CommandTimeoutSec -Transaction $Transaction
                    break
                }
                catch [System.Data.SqlClient.SqlException] {
                    if (-not $AutoIdentityInsert) {
                        throw
                    }

                    $attempt++
                    if ($attempt -gt 3) {
                        throw
                    }

                    $msg = $_.Exception.Message
                    $pattern = "Cannot insert explicit value for identity column in table '([^']+)' when IDENTITY_INSERT is set to OFF"
                    $m = [System.Text.RegularExpressions.Regex]::Match($msg, $pattern, [System.Text.RegularExpressions.RegexOptions]::IgnoreCase)
                    if (-not $m.Success) {
                        throw
                    }

                    $tableName = $m.Groups[1].Value
                    $targetRaw = Find-InsertTargetForTable -Batch $Batch -TableName $tableName
                    $target = ConvertTo-BracketedMultipartName -Name $targetRaw

                    if ($identityInsertTarget -and ($identityInsertTarget -ne $target)) {
                        Write-Verbose "Disabling IDENTITY_INSERT for $identityInsertTarget"
                        Invoke-MssqlBatch -Connection $Connection -Batch "SET IDENTITY_INSERT $identityInsertTarget OFF" -CommandTimeoutSec $CommandTimeoutSec -Transaction $Transaction
                        $identityInsertTarget = $null
                    }

                    if (-not $identityInsertTarget) {
                        Write-Verbose "Enabling IDENTITY_INSERT for $target (auto-retry due to identity insert error)"
                        Invoke-MssqlBatch -Connection $Connection -Batch "SET IDENTITY_INSERT $target ON" -CommandTimeoutSec $CommandTimeoutSec -Transaction $Transaction
                        $identityInsertTarget = $target
                    }

                    continue
                }
            }
        }
        finally {
            if ($identityInsertTarget) {
                try {
                    Write-Verbose "Disabling IDENTITY_INSERT for $identityInsertTarget"
                    Invoke-MssqlBatch -Connection $Connection -Batch "SET IDENTITY_INSERT $identityInsertTarget OFF" -CommandTimeoutSec $CommandTimeoutSec -Transaction $Transaction
                }
                catch {
                    Write-Warning "Failed to disable IDENTITY_INSERT for $identityInsertTarget. Error: $($_.Exception.Message)"
                }
            }
        }
    }

    function ConvertTo-BracketedSqlIdentifier {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Identifier
        )

        $trimmed = $Identifier.Trim()
        if ($trimmed.StartsWith('[', [System.StringComparison]::Ordinal) -and $trimmed.EndsWith(']', [System.StringComparison]::Ordinal)) {
            $trimmed = $trimmed.Substring(1, $trimmed.Length - 2)
        }

        $escaped = $trimmed.Replace(']', ']]')
        return "[$escaped]"
    }

    function Invoke-RecreateDatabase {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Data.SqlClient.SqlConnection]$MasterConnection,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$DatabaseName,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 86400)]
            [int]$CommandTimeoutSec
        )

        $dbLiteral = $DatabaseName.Replace("'", "''")
        $dbIdent = ConvertTo-BracketedSqlIdentifier -Identifier $DatabaseName

        $sql = @"
IF DB_ID(N'$dbLiteral') IS NOT NULL
BEGIN
    ALTER DATABASE $dbIdent SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE $dbIdent;
END
CREATE DATABASE $dbIdent;
"@

        Invoke-MssqlBatch -Connection $MasterConnection -Batch $sql -CommandTimeoutSec $CommandTimeoutSec
    }

    $repoRoot = Get-RepoRoot -StartDirectory $PSScriptRoot
    $defaultSqlDirectory = Join-Path -Path $repoRoot -ChildPath 'data\dumps\pvs\z1'

    if ([string]::IsNullOrWhiteSpace($SqlDirectory)) {
        $SqlDirectory = $defaultSqlDirectory
    }

    if (-not (Test-Path -LiteralPath $SqlDirectory -PathType Container)) {
        throw "SqlDirectory does not exist: '$SqlDirectory'"
    }

    if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
        $envConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'Process')
        if ([string]::IsNullOrWhiteSpace($envConnectionString)) {
            $envConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'User')
        }
        if ([string]::IsNullOrWhiteSpace($envConnectionString)) {
            $envConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'Machine')
        }

        if (-not [string]::IsNullOrWhiteSpace($envConnectionString)) {
            $ConnectionString = $envConnectionString
        }
    }

    if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
        if ($null -eq $Credential) {
            if ($WhatIfPreference) {
                Write-Verbose "WhatIf detected and no connection info provided. Skipping connection requirement for preview mode."
            }
            else {
                throw "No connection information provided. Use -ConnectionString, set env var Z1_MSSQL_CONNECTION_STRING, or provide -Credential (with -Server/-Port/-Database)."
            }
        }
        else {
            $ConnectionString = New-MssqlConnectionString -Server $Server -Port $Port -Database $Database -Credential $Credential
        }
    }

    if (-not [string]::IsNullOrWhiteSpace($ConnectionString)) {
        $ConnectionString = Normalize-ConnectionString -ConnectionString $ConnectionString
        $ConnectionString = Set-ConnectionStringDatabase -ConnectionString $ConnectionString -Database $Database
    }

    $fileNameSet = $null
    if ($null -ne $FileName -and $FileName.Count -gt 0) {
        $fileNameSet = New-Object 'System.Collections.Generic.HashSet[string]' ([System.StringComparer]::OrdinalIgnoreCase)
        foreach ($name in $FileName) {
            [void]$fileNameSet.Add($name)
        }
    }

    $excludeFileNameSet = $null
    if ($null -ne $ExcludeFileName -and $ExcludeFileName.Count -gt 0) {
        $excludeFileNameSet = New-Object 'System.Collections.Generic.HashSet[string]' ([System.StringComparer]::OrdinalIgnoreCase)
        foreach ($name in $ExcludeFileName) {
            [void]$excludeFileNameSet.Add($name)
        }
    }

    $sqlFiles = Get-ChildItem -LiteralPath $SqlDirectory -Filter '*.sql' -File -ErrorAction Stop | Sort-Object -Property Name

    if ($null -ne $fileNameSet) {
        $sqlFiles = $sqlFiles | Where-Object { $fileNameSet.Contains($_.Name) }
    }
    if ($null -ne $excludeFileNameSet) {
        $sqlFiles = $sqlFiles | Where-Object { -not $excludeFileNameSet.Contains($_.Name) }
    }

    $sqlFiles = @($sqlFiles)

    if ($sqlFiles.Count -eq 0) {
        throw "No SQL files found to import in '$SqlDirectory'."
    }

    if ($ResolveDependencies) {
        $sqlFiles = @(Resolve-SqlFileImportOrder -SqlFiles $sqlFiles)
    }

    $results = New-Object System.Collections.Generic.List[object]
    $failures = New-Object System.Collections.Generic.List[object]
    $startedAt = Get-Date
}

process {
    $connection = $null
    try {
        if (-not $WhatIfPreference) {
            if ($Force.IsPresent -or $EnsureDatabase.IsPresent) {
                $masterConnectionString = Set-ConnectionStringDatabase -ConnectionString $ConnectionString -Database 'master'
                $masterConnection = $null
                try {
                    if ($Force.IsPresent) {
                        $action = "Force re-create database (DROP and CREATE)"
                        if ($PSCmdlet.ShouldProcess($Database, $action)) {
                            Write-Verbose "Force re-creating database '$Database' (connecting to master)."
                            $masterConnection = Open-MssqlConnection -ConnectionString $masterConnectionString
                            Invoke-RecreateDatabase -MasterConnection $masterConnection -DatabaseName $Database -CommandTimeoutSec $CommandTimeoutSec
                        }
                    }
                    elseif ($EnsureDatabase.IsPresent) {
                        $action = "Ensure database exists"
                        if ($PSCmdlet.ShouldProcess($Database, $action)) {
                            Write-Verbose "Ensuring database '$Database' exists (connecting to master)."
                            $masterConnection = Open-MssqlConnection -ConnectionString $masterConnectionString
                            $escapedDb = $Database.Replace(']', ']]')
                            $ensureSql = "IF DB_ID(N'$escapedDb') IS NULL BEGIN CREATE DATABASE [$escapedDb]; END"
                            Invoke-MssqlBatch -Connection $masterConnection -Batch $ensureSql -CommandTimeoutSec $CommandTimeoutSec
                        }
                    }
                }
                finally {
                    if ($null -ne $masterConnection) {
                        $masterConnection.Dispose()
                    }
                }
            }

            Write-Verbose "Connecting to MSSQL with provided connection string (password redacted in logs)."
            $connection = Open-MssqlConnection -ConnectionString $ConnectionString
        }

        $total = [int]$sqlFiles.Count
        $index = 0

        foreach ($file in $sqlFiles) {
            $index++
            $fileStartedAt = Get-Date
            $activity = "Importing Z1 dumps into MSSQL"
            $status = "[$index/$total] $($file.Name)"
            Write-Progress -Activity $activity -Status $status -PercentComplete ([int](($index / [double]$total) * 100))

            if (-not $PSCmdlet.ShouldProcess($file.FullName, "Import SQL dump")) {
                continue
            }

            if ($WhatIfPreference) {
                continue
            }

            Write-Verbose "Reading SQL file: $($file.FullName)"
            $sqlText = Get-TextFromFile -LiteralPath $file.FullName
            if ($DeferCreateIndex) {
                $sqlText = Defer-CreateIndexBlocks -SqlText $sqlText
            }
            $batches = @(Split-SqlBatches -SqlText $sqlText)
            $batches = @(
                foreach ($batch in $batches) {
                    Split-LargeSqlBatch -Batch $batch -MaxBatchChars $MaxBatchChars
                }
            )

            if ($batches.Count -eq 0) {
                Write-Verbose "Skipping empty SQL file: $($file.Name)"
                continue
            }

            $transaction = $null
            $committed = $false
            $batchCount = [int]$batches.Count

            try {
                if ($UseTransaction.IsPresent) {
                    Write-Verbose "Beginning transaction for file: $($file.Name)"
                    $transaction = $connection.BeginTransaction()
                }

                for ($b = 0; $b -lt $batchCount; $b++) {
                    $batchNumber = $b + 1
                    Write-Verbose "Executing batch $batchNumber/$batchCount for file: $($file.Name)"
                    Invoke-MssqlBatchWithIdentityInsertHandling -Connection $connection -Batch $batches[$b] -CommandTimeoutSec $CommandTimeoutSec -Transaction $transaction -AutoIdentityInsert $AutoIdentityInsert
                }

                if ($null -ne $transaction) {
                    $transaction.Commit()
                    $committed = $true
                }

                $duration = (New-TimeSpan -Start $fileStartedAt -End (Get-Date))
                $results.Add([pscustomobject]@{
                    FileName        = $file.Name
                    FullName        = $file.FullName
                    BatchCount      = $batchCount
                    Duration        = $duration
                    Succeeded       = $true
                    ErrorMessage    = $null
                }) | Out-Null
            }
            catch {
                $err = $_
                if ($null -ne $transaction -and -not $committed) {
                    try {
                        Write-Warning "Rolling back transaction for file '$($file.Name)' due to error."
                        $transaction.Rollback()
                    }
                    catch {
                        Write-Warning "Rollback failed for file '$($file.Name)'. Error: $($_.Exception.Message)"
                    }
                }

                $duration = (New-TimeSpan -Start $fileStartedAt -End (Get-Date))
                $failureObj = [pscustomobject]@{
                    FileName        = $file.Name
                    FullName        = $file.FullName
                    BatchCount      = $batchCount
                    Duration        = $duration
                    Succeeded       = $false
                    ErrorMessage    = $err.Exception.Message
                }
                $failures.Add($failureObj) | Out-Null
                $results.Add($failureObj) | Out-Null

                if (-not $ContinueOnError.IsPresent) {
                    throw
                }
            }
            finally {
                if ($null -ne $transaction) {
                    $transaction.Dispose()
                }
            }
        }
    }
    finally {
        Write-Progress -Activity "Importing Z1 dumps into MSSQL" -Completed
        if ($null -ne $connection) {
            $connection.Dispose()
        }
    }
}

end {
    $endedAt = Get-Date
    $durationTotal = (New-TimeSpan -Start $startedAt -End $endedAt)

    [pscustomobject]@{
        SqlDirectory     = $SqlDirectory
        FileCount        = [int]$sqlFiles.Count
        ImportedCount    = [int](@($results | Where-Object { $_.Succeeded })).Count
        FailedCount      = [int]$failures.Count
        StartedAt        = $startedAt
        EndedAt          = $endedAt
        Duration         = $durationTotal
        Results          = $results
        Failures         = $failures
    }
}

```


</details>












































<br><br>
<br><br>

## Export

### SSMS

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




















<br><br>

---

<br><br>

## Docker

### Docker-compose

#### 2022

### `docker-compose.yml`

```yaml
networks:
  default:
    name: localdev
    external: true

volumes:
  # Docker-managed volume for MSSQL (works reliably on Windows/WSL2 + Linux)
  mssql-data:

services:
  # One-shot init container to fix permissions on the MSSQL volume for SQL Server 2022 non-root.
  # SQL Server runs as user `mssql` (UID 10001), so the data directory must be writable by 10001.
  mssql-init:
    image: alpine:3.19
    container_name: mssql-dev-init
    user: "0:0"
    command:
      - sh
      - -c
      - >
        mkdir -p /var/opt/mssql/data /var/opt/mssql/log /var/opt/mssql/secrets &&
        chown -R 10001:0 /var/opt/mssql &&
        chmod -R g=u /var/opt/mssql
    volumes:
      - mssql-data:/var/opt/mssql
    restart: "no"

  # One-shot post-start config for SQL Server memory tuning.
  # Why: Under heavy imports SQL Server can trigger memory pressure (Error 701) and close TCP connections.
  # This container runs once after MSSQL becomes healthy and sets `max server memory (MB)`.
  #
  # Reference: https://hub.docker.com/r/microsoft/mssql-server
  mssql-config:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-dev-config
    depends_on:
      mssql:
        condition: service_healthy
    environment:
      MSSQL_SA_PASSWORD: "Test1234!"
      # Default is intentionally moderate (safe-by-default for dev machines).
      # Override by setting MSSQL_MAX_SERVER_MEMORY_MB (e.g. in your shell env or a .env file).
      MSSQL_MAX_SERVER_MEMORY_MB: "${MSSQL_MAX_SERVER_MEMORY_MB:-6144}"
      # Import-friendly defaults (optional overrides)
      MSSQL_MAXDOP: "${MSSQL_MAXDOP:-4}"
      MSSQL_COST_THRESHOLD_FOR_PARALLELISM: "${MSSQL_COST_THRESHOLD_FOR_PARALLELISM:-50}"
    entrypoint:
      - /bin/bash
      - -lc
    command:
      - >
        /opt/mssql-tools18/bin/sqlcmd -S mssql -U sa -P "$${MSSQL_SA_PASSWORD}" -No -Q
        "EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
         EXEC sp_configure 'max server memory (MB)', $${MSSQL_MAX_SERVER_MEMORY_MB}; RECONFIGURE;
         EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;
         EXEC sp_configure 'max degree of parallelism', $${MSSQL_MAXDOP}; RECONFIGURE;
         EXEC sp_configure 'cost threshold for parallelism', $${MSSQL_COST_THRESHOLD_FOR_PARALLELISM}; RECONFIGURE;
         DBCC FREESYSTEMCACHE('SQL Plans');
         DBCC FREEPROCCACHE;"
    restart: "no"

  mssql:
    depends_on:
      mssql-init:
        condition: service_completed_successfully
    extends:
      file: services/mssql/service.yml
      service: mssql
    # Override volume strategy to ensure we *only* use the Docker-managed volume.
    volumes:
      - mssql-data:/var/opt/mssql
    # Memory settings (defaults are intentionally moderate / safe-by-default for dev machines).
    # Override by setting MSSQL_MEM_LIMIT / MSSQL_MEM_RESERVATION (e.g. in your shell env or a .env file).
    mem_limit: ${MSSQL_MEM_LIMIT:-8g}
    mem_reservation: ${MSSQL_MEM_RESERVATION:-4g}
```

### `services/mssql/service.yml`

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
      # Use a Docker-managed named volume by default (reliable on Windows/WSL2 + non-root MSSQL).
      # NOTE: The root compose file declares the volume and also runs a one-shot init container
      # that sets the correct ownership/permissions for SQL Server 2022 (user mssql, UID 10001).
      - mssql-data:/var/opt/mssql

    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P Test1234! -No -Q 'SELECT 1' || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```
