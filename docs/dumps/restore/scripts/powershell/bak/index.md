

#### Restore from bak file


```powershell
<#
.SYNOPSIS
    Restores the Z1 base database from a SQL Server .bak backup.
.DESCRIPTION
    Restores a SQL Server backup file (*.bak) into a local MSSQL instance (Docker or local SQL Server).
    This script is the "base" counterpart to the timer SQL-dump importer.

    Key features:
    - Supports WhatIf/Confirm via ShouldProcess
    - Can optionally copy the .bak into a running Docker container (default: mssql-dev)
    - Uses RESTORE FILELISTONLY to detect logical file names and generates WITH MOVE paths
    - Supports destructive overwrite via -Force
    - Object output summary for automation

    Connection options:
    - Provide -ConnectionString directly (recommended)
    - Or set environment variable Z1_MSSQL_CONNECTION_STRING

    Note:
    - A .bak is a binary backup. It is NOT executed like a .sql script.
    - SQL Server must be able to access the backup file path. For Docker, the script can copy it into the container.
.PARAMETER BackupPath
    Path to the SQL Server backup file (*.bak) on the host.
    Default resolves to: data\dumps\pvs\z1\base\bak\Z1_FULL_20250821.bak
.PARAMETER DatabaseName
    Target database name to restore. Default is 'Z1'.
.PARAMETER ConnectionString
    ADO.NET connection string used to connect to MSSQL. The script connects to master for restore operations.
    If not provided, the script uses Z1_MSSQL_CONNECTION_STRING.
.PARAMETER Server
    MSSQL host name or IP. Default is 'localhost'. Used when building a connection string from -Credential.
.PARAMETER Port
    MSSQL TCP port. Default is 1433. Used when building a connection string from -Credential.
.PARAMETER Credential
    SQL authentication credential (e.g. 'sa'). Used to build a connection string when -ConnectionString/env var are not provided.
.PARAMETER CommandTimeoutSec
    SQL command timeout for restore operations, in seconds. Default is 7200.
.PARAMETER UseDocker
    If true, the script will copy the backup into a Docker container and restore from the container path.
    Default is true.
.PARAMETER DockerContainerName
    Docker container name for MSSQL. Default is 'mssql-dev'.
.PARAMETER DockerBackupDirectory
    Directory inside the container used to store backups. Default is '/var/opt/mssql/backup'.
.PARAMETER Force
    DANGEROUS (destructive): Overwrite an existing database with the same name.
    Implements single-user with rollback immediate and RESTORE WITH REPLACE.
.EXAMPLE
    .\import-dumps-by-bak.ps1 -DatabaseName Z1 -Force -Confirm:$false -Verbose
.EXAMPLE
    .\import-dumps-by-bak.ps1 -BackupPath "C:\path\to\Z1_FULL_20250821.bak" -DatabaseName Z1 -Force -Confirm:$false -Verbose
#>
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'High')]
param(
    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$BackupPath,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$DatabaseName = 'Z1',

    [Parameter(Mandatory = $false)]
    [string]$ConnectionString,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$Server = 'localhost',

    [Parameter(Mandatory = $false)]
    [ValidateRange(1, 65535)]
    [int]$Port = 1433,

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [System.Management.Automation.PSCredential]$Credential,

    [Parameter(Mandatory = $false)]
    [ValidateRange(30, 86400)]
    [int]$CommandTimeoutSec = 7200,

    [Parameter(Mandatory = $false)]
    [ValidateNotNull()]
    [bool]$UseDocker = $true,

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$DockerContainerName = 'mssql-dev',

    [Parameter(Mandatory = $false)]
    [ValidateNotNullOrEmpty()]
    [string]$DockerBackupDirectory = '/var/opt/mssql/backup',

    [Parameter(Mandatory = $false)]
    [switch]$Force
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

            if (Test-Path -LiteralPath $packageJson -PathType Leaf) { return $current }
            if (Test-Path -LiteralPath $gitDir -PathType Container) { return $current }

            $parent = Split-Path -Path $current -Parent
            if ([string]::IsNullOrWhiteSpace($parent) -or ($parent -eq $current)) { break }
            $current = $parent
        }

        throw "Unable to determine repository root from '$StartDirectory'. Please pass -BackupPath explicitly."
    }

    function Normalize-ConnectionString {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$ConnectionString
        )

        # Some docs/tools use 'Encrypt=Optional'. System.Data.SqlClient expects a boolean.
        return [System.Text.RegularExpressions.Regex]::Replace(
            $ConnectionString,
            '(?i)(^|;)\s*Encrypt\s*=\s*Optional\s*(?=;|$)',
            '$1Encrypt=False'
        )
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
        return "Server=$Server,$Port;Database=$Database;User Id=$user;Password=$pwd;Encrypt=False;TrustServerCertificate=True;"
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
        if ([string]::IsNullOrWhiteSpace($trimmed)) { return "Database=$Database;" }
        return "$trimmed;Database=$Database;"
    }

    function Open-MssqlConnection {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$ConnectionString
        )

        $conn = New-Object System.Data.SqlClient.SqlConnection($ConnectionString)
        $conn.Open()
        return $conn
    }

    function Invoke-MssqlNonQuery {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Data.SqlClient.SqlConnection]$Connection,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Sql,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 86400)]
            [int]$CommandTimeoutSec
        )

        $cmd = $null
        try {
            $cmd = $Connection.CreateCommand()
            $cmd.CommandType = [System.Data.CommandType]::Text
            $cmd.CommandTimeout = $CommandTimeoutSec
            $cmd.CommandText = $Sql
            [void]$cmd.ExecuteNonQuery()
        }
        finally {
            if ($null -ne $cmd) { $cmd.Dispose() }
        }
    }

    function Invoke-MssqlQueryTable {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNull()]
            [System.Data.SqlClient.SqlConnection]$Connection,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$Sql,

            [Parameter(Mandatory = $true)]
            [ValidateRange(1, 86400)]
            [int]$CommandTimeoutSec
        )

        $cmd = $null
        $adapter = $null
        try {
            $cmd = $Connection.CreateCommand()
            $cmd.CommandType = [System.Data.CommandType]::Text
            $cmd.CommandTimeout = $CommandTimeoutSec
            $cmd.CommandText = $Sql

            $adapter = New-Object System.Data.SqlClient.SqlDataAdapter($cmd)
            $table = New-Object System.Data.DataTable
            [void]$adapter.Fill($table)
            # IMPORTANT: DataTable is enumerable; ensure we return it as a single object (not enumerated to DataRows).
            return ,$table
        }
        finally {
            if ($null -ne $adapter) { $adapter.Dispose() }
            if ($null -ne $cmd) { $cmd.Dispose() }
        }
    }

    function Assert-DockerAvailable {
        [CmdletBinding()]
        param()

        $docker = Get-Command -Name docker -ErrorAction SilentlyContinue
        if ($null -eq $docker) {
            throw "Docker CLI not found. Install Docker or run with -UseDocker:$false and ensure SQL Server can access the backup path."
        }
    }

    function Invoke-Docker {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string[]]$Arguments
        )

        function ConvertTo-WindowsCommandLineArg {
            param(
                [Parameter(Mandatory = $true)]
                [string]$Arg
            )

            if ($Arg -notmatch '[\s"]') {
                return $Arg
            }

            # Minimal quoting for CreateProcess command line:
            # wrap in double quotes and escape embedded quotes.
            $escaped = $Arg.Replace('"', '\"')
            return '"' + $escaped + '"'
        }

        $psi = New-Object System.Diagnostics.ProcessStartInfo
        $psi.FileName = 'docker'
        $psi.Arguments = ($Arguments | ForEach-Object { ConvertTo-WindowsCommandLineArg -Arg $_ }) -join ' '
        $psi.RedirectStandardOutput = $true
        $psi.RedirectStandardError = $true
        $psi.UseShellExecute = $false
        $psi.CreateNoWindow = $true

        $p = New-Object System.Diagnostics.Process
        $p.StartInfo = $psi
        [void]$p.Start()
        $stdout = $p.StandardOutput.ReadToEnd()
        $stderr = $p.StandardError.ReadToEnd()
        $p.WaitForExit()

        if ($p.ExitCode -ne 0) {
            throw "Docker command failed (exit $($p.ExitCode)). Args: $($psi.Arguments)`n$stderr"
        }

        return $stdout
    }

    function Ensure-BackupInDocker {
        [CmdletBinding()]
        param(
            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$DockerContainerName,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$DockerBackupDirectory,

            [Parameter(Mandatory = $true)]
            [ValidateNotNullOrEmpty()]
            [string]$HostBackupPath
        )

        Assert-DockerAvailable

        if (-not (Test-Path -LiteralPath $HostBackupPath -PathType Leaf)) {
            throw "BackupPath does not exist: '$HostBackupPath'"
        }

        $fileName = [System.IO.Path]::GetFileName($HostBackupPath)
        $containerBackupPath = ($DockerBackupDirectory.TrimEnd('/') + '/' + $fileName)

        # Ensure backup directory exists
        $mkdirCmd = "mkdir -p '$($DockerBackupDirectory.Replace("'", "''"))'"
        [void](Invoke-Docker -Arguments @('exec', $DockerContainerName, 'bash', '-lc', $mkdirCmd))

        # Copy backup file into container (overwrite)
        [void](Invoke-Docker -Arguments @('cp', $HostBackupPath, ("{0}:{1}" -f $DockerContainerName, $containerBackupPath)))

        return $containerBackupPath
    }

    $repoRoot = Get-RepoRoot -StartDirectory $PSScriptRoot
    $defaultBackupPath = Join-Path -Path $repoRoot -ChildPath 'data\dumps\pvs\z1\base\bak\Z1_FULL_20250821.bak'
    if ([string]::IsNullOrWhiteSpace($BackupPath)) {
        $BackupPath = $defaultBackupPath
    }

    if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
        $ConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'Process')
        if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
            $ConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'User')
        }
        if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
            $ConnectionString = [Environment]::GetEnvironmentVariable('Z1_MSSQL_CONNECTION_STRING', 'Machine')
        }
    }

    if ([string]::IsNullOrWhiteSpace($ConnectionString)) {
        if ($null -ne $Credential) {
            $ConnectionString = New-MssqlConnectionString -Server $Server -Port $Port -Database 'master' -Credential $Credential
        }
        elseif ($WhatIfPreference) {
            Write-Verbose "WhatIf detected and no connection info provided. Skipping connection requirement for preview mode."
        }
        else {
            throw "No connection information provided. Use -ConnectionString, set env var Z1_MSSQL_CONNECTION_STRING, or provide -Credential (with -Server/-Port)."
        }
    }

    if (-not [string]::IsNullOrWhiteSpace($ConnectionString)) {
        $ConnectionString = Normalize-ConnectionString -ConnectionString $ConnectionString
    }

    $startedAt = Get-Date
}

process {
    $masterConn = $null
    try {
        $masterCs = Set-ConnectionStringDatabase -ConnectionString $ConnectionString -Database 'master'
        Write-Verbose "Connecting to MSSQL (master) (password redacted in logs)."
        $masterConn = Open-MssqlConnection -ConnectionString $masterCs

        $backupForRestore = $BackupPath
        if ($UseDocker) {
            if ($PSCmdlet.ShouldProcess($DockerContainerName, "Copy backup into container")) {
                Write-Verbose "Copying backup into Docker container '$DockerContainerName'."
                $backupForRestore = Ensure-BackupInDocker -DockerContainerName $DockerContainerName -DockerBackupDirectory $DockerBackupDirectory -HostBackupPath $BackupPath
            }
        }

        $backupForRestoreSql = $backupForRestore.Replace("'", "''")
        $dbLiteral = $DatabaseName.Replace("'", "''")

        # Determine logical file names
        $fileListSql = "RESTORE FILELISTONLY FROM DISK = N'$backupForRestoreSql';"
        if (-not $PSCmdlet.ShouldProcess($DatabaseName, "Read backup file list")) {
            return
        }
        Write-Verbose "Reading backup file list (RESTORE FILELISTONLY)."
        $fileList = Invoke-MssqlQueryTable -Connection $masterConn -Sql $fileListSql -CommandTimeoutSec $CommandTimeoutSec
        if ($fileList.Rows.Count -eq 0) {
            throw "RESTORE FILELISTONLY returned no rows. Backup path may be inaccessible to SQL Server: '$backupForRestore'."
        }

        $moves = New-Object System.Collections.Generic.List[string]
        $dataIndex = 0
        $logIndex = 0

        foreach ($row in $fileList.Rows) {
            $logicalName = [string]$row['LogicalName']
            $type = [string]$row['Type']

            if ([string]::IsNullOrWhiteSpace($logicalName) -or [string]::IsNullOrWhiteSpace($type)) {
                continue
            }

            if ($type -eq 'D') {
                $dataIndex++
                $targetPath = "/var/opt/mssql/data/$DatabaseName" + ($(if ($dataIndex -gt 1) { "_$dataIndex" } else { "" })) + ".mdf"
                $moves.Add(("MOVE N'{0}' TO N'{1}'" -f $logicalName.Replace("'", "''"), $targetPath.Replace("'", "''"))) | Out-Null
            }
            elseif ($type -eq 'L') {
                $logIndex++
                $targetPath = "/var/opt/mssql/data/$DatabaseName" + ($(if ($logIndex -gt 1) { "_log_$logIndex" } else { "_log" })) + ".ldf"
                $moves.Add(("MOVE N'{0}' TO N'{1}'" -f $logicalName.Replace("'", "''"), $targetPath.Replace("'", "''"))) | Out-Null
            }
        }

        if ($moves.Count -eq 0) {
            throw "Could not determine logical data/log files from backup."
        }

        $withParts = New-Object System.Collections.Generic.List[string]
        foreach ($m in $moves) { $withParts.Add($m) | Out-Null }
        $withParts.Add("STATS = 5") | Out-Null

        if ($Force.IsPresent) {
            $withParts.Add("REPLACE") | Out-Null
        }

        # Ensure exclusive access if force
        if ($Force.IsPresent) {
            if ($PSCmdlet.ShouldProcess($DatabaseName, "Set SINGLE_USER with rollback immediate")) {
                Write-Verbose "Setting database '$DatabaseName' to SINGLE_USER WITH ROLLBACK IMMEDIATE."
                $dbIdent = ('[' + $DatabaseName.Replace(']', ']]') + ']')
                Invoke-MssqlNonQuery -Connection $masterConn -Sql "IF DB_ID(N'$dbLiteral') IS NOT NULL BEGIN ALTER DATABASE $dbIdent SET SINGLE_USER WITH ROLLBACK IMMEDIATE; END" -CommandTimeoutSec $CommandTimeoutSec
            }
        }

        $restoreSql = @"
RESTORE DATABASE [$($DatabaseName.Replace(']', ']]'))]
FROM DISK = N'$backupForRestoreSql'
WITH $(($withParts -join ",`r`n     "));
"@

        if (-not $PSCmdlet.ShouldProcess($DatabaseName, "Restore database from backup")) {
            return
        }
        Write-Verbose "Restoring database '$DatabaseName' from backup."
        Invoke-MssqlNonQuery -Connection $masterConn -Sql $restoreSql -CommandTimeoutSec $CommandTimeoutSec

        # Back to multi-user (best-effort)
        if ($PSCmdlet.ShouldProcess($DatabaseName, "Set MULTI_USER")) {
            try {
                $dbIdent = ('[' + $DatabaseName.Replace(']', ']]') + ']')
                Invoke-MssqlNonQuery -Connection $masterConn -Sql "ALTER DATABASE $dbIdent SET MULTI_USER" -CommandTimeoutSec $CommandTimeoutSec
            }
            catch {
                Write-Warning "Failed to set MULTI_USER for '$DatabaseName'. Error: $($_.Exception.Message)"
            }
        }
    }
    finally {
        if ($null -ne $masterConn) { $masterConn.Dispose() }
    }
}

end {
    $endedAt = Get-Date
    [pscustomobject]@{
        BackupPath           = $BackupPath
        DatabaseName         = $DatabaseName
        UseDocker            = $UseDocker
        DockerContainerName  = $DockerContainerName
        DockerBackupDirectory= $DockerBackupDirectory
        Force                = [bool]$Force.IsPresent
        StartedAt            = $startedAt
        EndedAt              = $endedAt
        Duration             = (New-TimeSpan -Start $startedAt -End $endedAt)
    }
}


```
