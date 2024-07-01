---
title: Powershell Challenge dotnetSDK
description: Downloads the .NET SDK, check the downloaded file's signature and issuer are valid. Check with VirusTotal API using the hash value of the downloaded file. Install the SDK silently, log all installed SDK versions, and then finally uninstall all the previous versions. 
author: BW
date: 2024-01-20 10:00:00 - 0500
categories: [Scripting, Powershell]
tags: [scripting]     # TAG names should always be lowercase
---

### Powershell Challenge

Had some trouble with the last part of this script near the end of my 2 hour time constraint - uninstalling all previous versions. Sometimes I overthink a problem, and create a complex function when a simple approach would have been better and actually work. Otherwise, this script is complete and works as intended. And it was a lot of fun doing it!

What this script essentially does is download the .NET SDK, check the downloaded file's signature and issuer are valid. Check with VirusTotal API using the hash value of the downloaded file. Install the SDK silently, log all installed SDK versions, and then finally uninstall all the previous versions. 

```powershell
<#
.NOTES

Name: DotnetSDK.ps1
Author: B. Werkman
Version: 1.1
DateCreated: 19 Jan 24
#>

# Download .NET SDK

$installDir = "C:\install"
$downloadUrl = "https://download.visualstudio.microsoft.com/download/pr/cb56b18a-e2a6-4f24-be1d-fc4f023c9cc8/be3822e20b990cf180bb94ea8fbc42fe/dotnet-sdk-8.0.101-win-x64.exe"
$filePath = Join-Path -Path $installDir -ChildPath "dotnet-sdk-8.0.101-win-x64.exe"

# Define log file path with current date, time, and hostname

$currentDateTime = Get-Date -Format "yyyyMMddHHmmss"
$hostName = $env:COMPUTERNAME
$logPath = "C:\install\$($currentDateTime)_$($hostName).log"

# Create directory if it doesn't exist

if (!(Test-Path -Path $installDir)) {
    New-Item -ItemType Directory -Path $installDir

}

# Function to write log

function Write-Log {
    param (
        [Parameter(ValueFromPipeline=$true)]
        [string]$message
    )
    Process {
        "$((Get-Date).ToString()) : $message" | Out-File -FilePath $logPath -Append
    }
}

# Download file
Invoke-WebRequest -Uri $downloadUrl -OutFile $filePath

# Validate Authenticode signature and log if invalid
$signature = Get-AuthenticodeSignature -FilePath $filePath

$expectedIssuer = "CN=Microsoft Code Signing PCA 2011, O=Microsoft Corporation, L=Redmond, S=Washington, C=US"


# Check the signature status and issuer

if ($signature.Status -ne "Valid" -or $signature.SignerCertificate.Issuer -ne $expectedIssuer) {
    $logContent = "Invalid signature or issuer.`n"
    $logContent += "File Path: $filePath`n"
    $logContent += "Signature Status: $($signature.Status)`n"
    $logContent += "Issuer: $($signature.SignerCertificate.Issuer)`n"
    $logContent | Write-Log
    exit

}

# Check with VirusTotal
$apiKey = "3f50cb994f2859e6a81a14cc751df6aab11456e08405afb393f035f62456a373"
$fileHash = (Get-FileHash -Path $filePath -Algorithm SHA256).Hash
$virusTotalUrl = "https://www.virustotal.com/api/v3/files/$fileHash"

$headers = @{
    "x-apikey" = $apiKey
}

try {
$response = Invoke-RestMethod -Uri $virusTotalUrl -Method Get -Headers $headers -ErrorAction Stop
} catch {
    "Failed to get a response from VirusTotal. Error: $_" | Write-Log
    exit
}

# Check for bad reputation and log details

if ($response.data.attributes.reputation -lt 0) {
    $logContent = "File has a bad reputation on VirusTotal.`n"
    $logContent += "File Path: $filePath`n"
    $logContent += "VirusTotal Response: $($response | ConvertTo-Json -Depth 5)`n"
    $logContent | Write-Log
    exit
}

# Install .NET SDK
Start-Process -FilePath $filePath -ArgumentList "/quiet /norestart" -Wait

# Validate Installation
$dotNetPath = "${env:ProgramFiles}\dotnet\sdk\8.0.101" # Path where .NET SDK is expected to be installed

$installationValid = $false
if (Test-Path -Path $dotNetPath) {
    $installedVersions = Get-ChildItem -Path "${env:ProgramFiles}\dotnet\sdk\" | Select-Object -ExpandProperty Name
    if ("8.0.101" -in $installedVersions) {
        $installationValid = $true
    }
}

if (-not $installationValid) {
    "Installation of .NET SDK 8.0.101 failed or was not verified." | Write-Log
    exit
}

# Enumerate and Uninstall Previous Versions of .NET SDK
$targetVersion = "8.0.101"

$dotNetSdks = Get-ChildItem -Path "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall", "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" |
    Get-ItemProperty |
    Where-Object { $_.DisplayName -like "*.NET SDK*" -or $_.DisplayName -like "*.NET Core SDK*" }
    foreach ($sdk in $dotNetSdks) {
        Write-Log "Found SDK: $($sdk.DisplayName)"

       # Check if the DisplayName contains the target version

        if ($sdk.DisplayName -notlike "*$targetVersion*") {
            if ($sdk.UninstallString) {
                Write-Log "Uninstalling $($sdk.DisplayName)"
                Start-Process -FilePath "cmd.exe" -ArgumentList "/c", $($sdk.UninstallString + " /quiet /norestart") -Wait
                Write-Log "$($sdk.DisplayName) uninstalled"
            }
            else {
                Write-Log "No uninstall string found for $($sdk.DisplayName)"
            }
        }
        else {
            Write-Log "Skipping $($sdk.DisplayName), as it is the target version or version parsing failed"
        }
    }
```
{: file='dotnetSDK.ps1'}
