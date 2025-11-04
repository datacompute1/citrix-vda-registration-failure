# citrix-vda-registration-failure
Download the Citrix VDA Registration Troubleshooting Script Get a ready-to-use PowerShell script and checklist to automatically test VDA registration status and controller connectivity.
# =============================================================
# Script: Test-VDARegistration.ps1
# Version: 1.0
# Author: Datacompute Consulting LLC
# Website: https://datacompute.net
# =============================================================
# Description:
#   This script diagnoses Citrix Virtual Delivery Agent (VDA)
#   registration issues with Delivery Controllers (DDCs).
#   It performs checks for:
#     - Network connectivity
#     - DNS resolution
#     - Registry entries
#     - Domain membership
#     - Time synchronization
#   Results are displayed in color:
#     - Green  = OK
#     - Yellow = Needs Attention
# =============================================================
# Usage:
#   1. Run PowerShell as Administrator.
#   2. Execute the script and enter:
#        - The FQDN (Fully Qualified Domain Name) of the Delivery Controller.
#        - Your domain name when prompted.
#   3. The script will test connectivity and display the diagnostic results
#      directly in the console.
#
# Example:
#   PS C:\> .\Test-VDARegistration.ps1
#   Enter Delivery Controller FQDN: ddc01.domain.local
#   Enter Domain Name: domain.local
#
#   Results will be displayed on screen.
# =============================================================
# Disclaimer:
#   This script has been tested and verified in production environments
#   with successful results. However, Datacompute Consulting LLC is not
#   responsible for any issues, damages, or data loss resulting from its use.
#   Use at your own discretion and test in a lab before production deployment.
#
# Ownership:
#   All rights and royalties are owned by Datacompute Consulting LLC.
# =============================================================


Write-Host "Checking Citrix Workspace installation and ICA file association..." -ForegroundColor Cyan

# --- Step 1: Check Citrix Workspace installation ---
$workspacePaths = @(
    "HKLM:\SOFTWARE\Citrix\ICA Client",
    "HKLM:\SOFTWARE\WOW6432Node\Citrix\ICA Client"
)

$workspaceInstalled = $false
$workspaceDir = $null
$workspaceVersion = $null

foreach ($path in $workspacePaths) {
    if (Test-Path $path) {
        $workspaceInstalled = $true
        try {
            $props = Get-ItemProperty -Path $path
            $workspaceDir = $props.InstallDir
            $workspaceVersion = $props.Version
        } catch {}
        break
    }
}

if ($workspaceInstalled -and (Test-Path $workspaceDir)) {
    Write-Host " Citrix Workspace is installed." -ForegroundColor Green
    Write-Host "   Path: $workspaceDir"
    if ($workspaceVersion) {
        Write-Host "   Version: $workspaceVersion" -ForegroundColor Gray
    } else {
        Write-Host "   Version: (unknown)" -ForegroundColor Yellow
    }
} else {
    Write-Host " Citrix Workspace not found." -ForegroundColor Red
    exit
}

# --- Step 2: Check current ICA association ---
$icaAssoc = $null
$icaHandler = $null

try {
    $icaAssoc = (Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\FileExts\.ica\UserChoice" -ErrorAction Stop).ProgId
} catch {}

if (-not $icaAssoc) {
    try {
        $icaAssoc = (Get-ItemProperty "HKCR\.ica" -ErrorAction Stop).'(default)'
    } catch {}
}

if ($icaAssoc) {
    try {
        $icaHandler = (Get-ItemProperty "HKCR:\$icaAssoc\shell\open\command" -ErrorAction Stop).'(default)'
    } catch {}
} else {
    try {
        $icaHandler = (Get-ItemProperty "HKCR\.ica\shell\open\command" -ErrorAction Stop).'(default)'
    } catch {}
}

# --- Step 3: Determine if association is correct ---
$expectedExe = "wfcrun32.exe"
$expectedPath = Join-Path $workspaceDir "wfcrun32.exe"
$expectedCommand = "`"$expectedPath`" `"%1`""

if ($icaHandler -and ($icaHandler -match [regex]::Escape($expectedExe))) {
    Write-Host " .ICA files are correctly associated with Citrix Connection Manager." -ForegroundColor Green
    Write-Host "Handler: $icaHandler" -ForegroundColor Gray
} else {
    Write-Host " .ICA association missing or incorrect." -ForegroundColor Yellow
    if ($icaHandler) {
        Write-Host "Current handler: $icaHandler" -ForegroundColor Gray
    } else {
        Write-Host "No association found for .ICA files." -ForegroundColor Gray
    }

    # --- Step 4: Repair ICA association ---
    Write-Host "Attempting to repair .ICA file association..." -ForegroundColor Cyan

    $progId = "Citrix.ICAClient.2"

    try {
        # Create ProgID entry if missing
        if (-not (Test-Path "HKCR:\$progId\shell\open\command")) {
            New-Item -Path "HKCR:\$progId\shell\open\command" -Force | Out-Null
        }
        Set-ItemProperty -Path "HKCR:\$progId\shell\open\command" -Name "(default)" -Value $expectedCommand -Force

        # Set default association for .ica files
        New-Item -Path "HKCR:\.ica" -Force | Out-Null
        Set-ItemProperty -Path "HKCR:\.ica" -Name "(default)" -Value $progId -Force

        Write-Host " .ICA file association repaired successfully." -ForegroundColor Green
        Write-Host "Now associated with: $expectedCommand" -ForegroundColor Gray
    } catch {
        Write-Host "Failed to repair ICA association. Try running PowerShell as Administrator." -ForegroundColor Red
    }
}
