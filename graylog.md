# Graylog

## Aktualizacja Sidecar w Windows

Skrypt Powershell przydatny przy aktualizacji Sidecar w Windows, pobiera aktualną wersję, wyłącza procesy i uruchamia instalator. Czasami pomimo wyłączenia procesów, pojawia się komunikat o błędzie z tym związanym, należy po prostu kliknąć "Retry", a instalacja przebiegnie poprawnie.

```powershell
<# check latest version
$apiUrl = "https://api.github.com/repos/Graylog2/collector-sidecar/releases/latest"
$latestRelease = Invoke-RestMethod -Uri $apiUrl -UseBasicParsing
Write-Host "$($latestRelease.tag_name)"
#>

$SidecarVersion = '1.5.1'

$SidecarRepository = 'https://github.com/Graylog2/collector-sidecar/releases/download/' + $SidecarVersion + '/'
$SidecarFile = 'graylog_sidecar_installer_' + $SidecarVersion + '-1.exe'
$SidecarLink = $SidecarRepository + $SidecarFile
$UserDownloadFolder = [Environment]::GetFolderPath('UserProfile') + '\Downloads'
$InstallerOutput = $UserDownloadFolder + '\' + $SidecarFile

Write-Host "Try to download Graylog Sidecar, version $SidecarVersion"
if (Test-Path -Path $InstallerOutput -PathType Leaf) {
    Write-Host "Already downloaded"
} else {
    try {
        Invoke-WebRequest -Uri $SidecarLink -OutFile $InstallerOutput
        Write-Host "File downloaded successfully to $InstallerOutput" -ForegroundColor Green
    } catch {
        Write-Host "Something goes wrong" -ForegroundColor Red
        exit(0)
    }
}

Write-Host "Copy configuration file"
@('C:\Program Files\Graylog\sidecar\CA.pem', 'C:\Program Files\Graylog\sidecar\sidecar.yml') | ForEach-Object {
    if (Test-Path -Path $_ -PathType Leaf) {
        $fileName = Split-Path -Path $_ -Leaf
        if (Test-Path -Path (Join-Path -Path $UserDownloadFolder -ChildPath ($fileName + '.old')) -PathType Leaf) {
            Write-Host "Backup exist, remove it first" -ForegroundColor Red
            exit(0)
        }
        Copy-Item -Path $_ -Destination (Join-Path -Path $UserDownloadFolder -ChildPath ($fileName + '.old'))
        Write-Host "Configuration file saved" -ForegroundColor Green
    } else {
        Write-Host "Configuration file not found"
    }
}

@('graylog-sidecar.exe', 'winlogbeat.exe') | ForEach-Object {
    Get-Process -Name ([System.IO.Path]::GetFileNameWithoutExtension($_)) -ErrorAction SilentlyContinue | ForEach-Object {
        try {
            Stop-Process -Id $_.Id -Force
            Write-Host "Process $($_.Name) (PID: $($_.Id)) stopped successfully" -ForegroundColor Green
        } catch {
            Write-Host "Failed to stop process $($_.Name): $_" -ForegroundColor Red
        }
    }

    if (-not (Get-Process -Name ([System.IO.Path]::GetFileNameWithoutExtension($_)) -ErrorAction SilentlyContinue)) {
        Write-Host "Process $_ not found" -ForegroundColor Yellow
    }
}


$StartProcess = Start-Process -FilePath $InstallerOutput -PassThru -Wait -NoNewWindow
if ($StartProcess.ExitCode -eq 0) {
    Write-Host "Installation completed successfully" -ForegroundColor Green
} else {
    Write-Host "Installation failed with exit code $($StartProcess.ExitCode)" -ForegroundColor Red
}
```
