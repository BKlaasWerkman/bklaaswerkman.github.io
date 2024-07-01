---
title: Get-MFAStatus.ps1
description: This script will get the Azure MFA Status of users. You can query all the users, admins only or a single user. It will return the MFA Status, MFA type and registered devices.
author: BW
date: 2024-01-06 12:00:00 - 0500
categories: [Scripting, Powershell]
tags: [scripting]     # TAG names should always be lowercase
---

### Get-MFAStatus.ps1 

```powershell
<#
.DESCRIPTION
  This script will get the Azure MFA Status of users. You can query all the users, admins only or a single user.
  It will return the MFA Status, MFA type and registered devices.

.NOTES
  Name: Get-MFAStatus
  Author: B. Werkman
  Version: 1.4
  DateCreated: Jan 2024
  Changes: This script uses the newer Microsoft Graph Powershell Module instead of the Deprecated Msol Module which will be officially retired in March of 2024.
            No need to specify a path, script will automatically create a subdirectory named "Reports" and output the csv and excel file to it using filename format of MFAStatus-MMM-dd-yyyy.csv and open it for you.

.EXAMPLE
  Get-MFAStatus
  Get the MFA Status of all enabled and licensed users and check if there are an admin or not
  Get-MFAStatus -UserPrincipalName 'john.doe@contoso.com','jane.doe@contoso.com'
  Get the MFA Status for the users John Doe and Jane Doe
  Get-MFAStatus -withOutMFAOnly
  Get only the licensed and enabled users that don't have MFA enabled
  Get-MFAStatus -adminsOnly
  Get the MFA Status of the admins only
  Get-MgUser -Filter "country eq 'Netherlands'" | ForEach-Object { Get-MFAStatus -UserPrincipalName $_.UserPrincipalName }
  Get the MFA status for all users in the Country The Netherlands. You can use a similar approach to run this
  for a department only.
#>

[CmdletBinding(DefaultParameterSetName="Default")]

param(
  [Parameter(
    Mandatory = $false,
    ParameterSetName  = "UserPrincipalName",
    HelpMessage = "Enter a single UserPrincipalName or a comma separted list of UserPrincipalNames",
    Position = 0
    )]

  [string[]]$UserPrincipalName,
  [Parameter(
    Mandatory = $false,
    ValueFromPipeline = $false,
    ParameterSetName  = "AdminsOnly"
  )]

  # Get only the users that are an admin
  [switch]$adminsOnly = $false,
  [Parameter(
    Mandatory         = $false,
    ValueFromPipeline = $false,
    ParameterSetName  = "Licensed"
  )]

  # Check only the MFA status of users that have license
  [switch]$IsLicensed = $true,
  [Parameter(
    Mandatory         = $false,
    ValueFromPipeline = $true,
    ValueFromPipelineByPropertyName = $true,
    ParameterSetName  = "withOutMFAOnly"
  )]

  # Get only the users that don't have MFA enabled
  [switch]$withOutMFAOnly = $false,
  [Parameter(
    Mandatory         = $false,
    ValueFromPipeline = $false
  )]

  # Check if a user is an admin. Set to $false to skip the check
  [switch]$listAdmins = $true,
  [Parameter(
    Mandatory = $false,
    HelpMessage = "Enter path to save the CSV file"
  )]
  [string]$path = ".\MFAStatus-$((Get-Date -format "MMM-dd-yyyy").ToString()).csv"
)

Function Install-ImportExcel {
  # Check if MS Graph module is installed
  if (-not(Get-InstalledModule ImportExcel)) {
   Write-Host "Excel module not found" -ForegroundColor Black -BackgroundColor Yellow
   $install = Read-Host "Do you want to install the Excel Module?"

   if ($install -match "[yY]") {
     Install-Module ImportExcel -Repository PSGallery -Scope CurrentUser -AllowClobber -Force
   }else{
     Write-Host "Excel module is required." -ForegroundColor Black -BackgroundColor Yellow
     exit
   }
  }
 }

Function ConnectTo-MgGraph {
  # Check if MS Graph module is installed
  if (-not(Get-InstalledModule Microsoft.Graph)) {
    Write-Host "Microsoft Graph module not found" -ForegroundColor Black -BackgroundColor Yellow
    $install = Read-Host "Do you want to install the Microsoft Graph Module?"
    if ($install -match "[yY]") {
      Install-Module Microsoft.Graph -Repository PSGallery -Scope CurrentUser -AllowClobber -Force
    }else{
      Write-Host "Microsoft Graph module is required." -ForegroundColor Black -BackgroundColor Yellow
      exit
    }
  }

  # Connect to Graph
  Write-Host "Connecting to Microsoft Graph" -ForegroundColor Cyan
  Connect-MgGraph -Scopes "User.Read.All, UserAuthenticationMethod.Read.All, Directory.Read.All" -NoWelcome
}
Function Get-Admins{
  <#
  .SYNOPSIS
    Get all user with an Admin role
  #>
  process{
    $admins = Get-MgDirectoryRole | Select-Object DisplayName, Id |
                ForEach-Object{$role = $_.DisplayName; Get-MgDirectoryRoleMember -DirectoryRoleId $_.id |
                  Where-Object {$_.AdditionalProperties."@odata.type" -eq "#microsoft.graph.user"} |
                  ForEach-Object {Get-MgUser -userid $_.id }
                } |
                Select-Object @{Name="Role"; Expression = {$role}}, DisplayName, UserPrincipalName, Mail, Id | Sort-Object -Property Mail -Unique
    return $admins
  }
}

Function Get-Users {
  <#
  .SYNOPSIS
    Get users from the requested DN
  #>
  process{
    # Set the properties to retrieve
    $select = @(
      'id',
      'DisplayName',
      'userprincipalname',
      'mail'
    )
    $properties = $select + "AssignedLicenses"
    # Get enabled, disabled or both users
    switch ($enabled)
    {
      "true" {$filter = "AccountEnabled eq true and UserType eq 'member'"}
      "false" {$filter = "AccountEnabled eq false and UserType eq 'member'"}
      "both" {$filter = "UserType eq 'member'"}
    }

    # Check if UserPrincipalName(s) are given
    if ($UserPrincipalName) {
      Write-host "Get users by name" -ForegroundColor Cyan
      $users = @()
      foreach ($user in $UserPrincipalName)
      {
        try {
          $users += Get-MgUser -UserId $user -Property $properties | Select-Object $select -ErrorAction Stop
        }
        catch {
          [PSCustomObject]@{
            DisplayName       = " - Not found"
            UserPrincipalName = $User
            isAdmin           = $null
            MFAEnabled        = $null
          }
        }
      }
    }elseif($adminsOnly)
    {
      Write-host "Get admins only" -ForegroundColor Cyan
      $users = @()
      foreach ($admin in $admins) {
        $users += Get-MgUser -UserId $admin.UserPrincipalName -Property $properties | Select-Object $select
      }
    }else
    {
      if ($IsLicensed) {
        # Get only licensed users
        $users = Get-MgUser -Filter $filter -Property $properties -all | Where-Object {($_.AssignedLicenses).count -gt 0} | Select-Object $select
      }else{
        $users = Get-MgUser -Filter $filter -Property $properties -all | Select-Object $select
      }
    }
    return $users
  }
}
Function Get-MFAMethods {
  <#
    .SYNOPSIS
      Get the MFA status of the user
  #>
  param(
    [Parameter(Mandatory = $true)] $userId
  )
  process{
    # Get MFA details for each user
    [array]$mfaData = Get-MgUserAuthenticationMethod -UserId $userId

    # Create MFA details object
    $mfaMethods  = [PSCustomObject][Ordered]@{
      status            = "NA"
      authApp           = "NA"
      phoneAuth         = "NA"
      fido              = "NA"
      helloForBusiness  = "NA"
      helloForBusinessCount = 0
      emailAuth         = "NA"
      tempPass          = "NA"
      passwordLess      = "NA"
      softwareAuth      = "NA"
      authDevice        = "NA"
      authPhoneNr       = "NA"
      SSPREmail         = "NA"
    }

    ForEach ($method in $mfaData) {
        Switch ($method.AdditionalProperties["@odata.type"]) {
          "#microsoft.graph.microsoftAuthenticatorAuthenticationMethod"  {
            # Microsoft Authenticator App
            $mfaMethods.authApp = $true
            $mfaMethods.authDevice += $method.AdditionalProperties["displayName"]
            $mfaMethods.status = "enabled"
          }
          "#microsoft.graph.phoneAuthenticationMethod"                  {
            # Phone authentication
            $mfaMethods.phoneAuth = $true
            $mfaMethods.authPhoneNr = $method.AdditionalProperties["phoneType", "phoneNumber"] -join ' '
            $mfaMethods.status = "enabled"
          }

          "#microsoft.graph.fido2AuthenticationMethod"                   {
            # FIDO2 key
            $mfaMethods.fido = $true
            $fifoDetails = $method.AdditionalProperties["model"]
            $mfaMethods.status = "enabled"
          }
          "#microsoft.graph.passwordAuthenticationMethod"                {
            # Password
            # When only the password is set, then MFA is disabled.
            if ($mfaMethods.status -ne "enabled") {$mfaMethods.status = "disabled"}
          }

          "#microsoft.graph.windowsHelloForBusinessAuthenticationMethod" {
            # Windows Hello
            $mfaMethods.helloForBusiness = $true
            $helloForBusinessDetails = $method.AdditionalProperties["displayName"]
            $mfaMethods.status = "enabled"
            $mfaMethods.helloForBusinessCount++
          }

          "#microsoft.graph.emailAuthenticationMethod"                   {
            # Email Authentication
            $mfaMethods.emailAuth =  $true
            $mfaMethods.SSPREmail = $method.AdditionalProperties["emailAddress"]
            $mfaMethods.status = "enabled"
          }              
          "microsoft.graph.temporaryAccessPassAuthenticationMethod"    {
            # Temporary Access pass
            $mfaMethods.tempPass = $true
            $tempPassDetails = $method.AdditionalProperties["lifetimeInMinutes"]
            $mfaMethods.status = "enabled"
          }
          "#microsoft.graph.passwordlessMicrosoftAuthenticatorAuthenticationMethod" {
            # Passwordless
            $mfaMethods.passwordLess = $true
            $passwordLessDetails = $method.AdditionalProperties["displayName"]
            $mfaMethods.status = "enabled"
          }
          "#microsoft.graph.softwareOathAuthenticationMethod" {
            # ThirdPartyAuthenticator
            $mfaMethods.softwareAuth = $true
            $mfaMethods.status = "enabled"
          }
        }
    }
    Return $mfaMethods
  }
}

Function Get-Manager {
  <#
    .SYNOPSIS
      Get the manager users
  #>
  param(
    [Parameter(Mandatory = $true)] $userId
  )
  process {
    $manager = Get-MgUser -UserId $userId -ExpandProperty manager | Select-Object @{Name = 'name'; Expression = {$_.Manager.AdditionalProperties.displayName}}
    return $manager.name
  }
}
Function Get-MFAStatusUsers {
  <#
    .SYNOPSIS
      Get all AD users
  #>
  process {
    Write-Host "Collecting users" -ForegroundColor Cyan

    # Collect users
    $users = Get-Users
   
    Write-Host "Processing" $users.count "users" -ForegroundColor Cyan

    # Collect and loop through all users
    $users | ForEach-Object {
      $mfaMethods = Get-MFAMethods -userId $_.id
      $manager = Get-Manager -userId $_.id
       $uri = "https://graph.microsoft.com/beta/users/$($_.id)/authentication/signInPreferences"
       $mfaPreferredMethod = Invoke-MgGraphRequest -uri $uri -Method GET
       if ($null -eq ($mfaPreferredMethod.userPreferredMethodForSecondaryAuthentication)) {
        # When an MFA is configured by the user, then there is alway a preferred method
        # So if the preferred method is empty, then we can assume that MFA isn't configured
        # by the user
        $mfaMethods.status = "disabled"
       }

      if ($withOutMFAOnly) {
        if ($mfaMethods.status -eq "disabled") {
          [PSCustomObject]@{
            "Name" = $_.DisplayName
            Emailaddress = $_.mail
            UserPrincipalName = $_.UserPrincipalName
            isAdmin = if ($listAdmins -and ($admins.UserPrincipalName -match $_.UserPrincipalName)) {$true} else {"no"}
            MFAEnabled        = $false
            "Phone_number" = $mfaMethods.authPhoneNr
            "Email_for_SSPR" = $mfaMethods.SSPREmail
          }
        }
      }else{
        [pscustomobject]@{
          "Name" = $_.DisplayName
          Emailaddress = $_.mail
          UserPrincipalName = $_.UserPrincipalName
          isAdmin = if ($listAdmins -and ($admins.UserPrincipalName -match $_.UserPrincipalName)) {$true} else {"no"}
          "MFA_Status" = $mfaMethods.status
          "MFA_Preferred_method" = $mfaPreferredMethod.userPreferredMethodForSecondaryAuthentication
          "Phone_Authentication" = $mfaMethods.phoneAuth
          "Authenticator_App" = $mfaMethods.authApp
          "Passwordless" = $mfaMethods.passwordLess
          "Hello_for_Business" = $mfaMethods.helloForBusiness
          "FIDO2_Security_Key" = $mfaMethods.fido
          "Temporary_Access_Pass" = $mfaMethods.tempPass
          "Authenticator_device" = $mfaMethods.authDevice
          "Phone_number" = $mfaMethods.authPhoneNr
          "Email_for_SSPR" = $mfaMethods.SSPREmail
          "Manager" = $manager
        }
      }
    }
  }
}
# Install Excel module
Install-ImportExcel

# Connect to Graph function
ConnectTo-MgGraph

# Get Admins
# Get all users with admin role
$admins = $null
if (($listAdmins) -or ($adminsOnly)) {
  $admins = Get-Admins
}
$mfaStatusData = Get-MFAStatusUsers

# Checks if Reports folder exists and if not then creates it
$ReportPath = Join-Path -Path $PSScriptRoot -ChildPath "Reports"

if (-not(Test-Path -Path $ReportPath)){
  Write-Host "Reports folder doesn't exist, creating folder in $PSScriptRoot" -ForegroundColor Cyan
  New-Item -Path $ReportPath -ItemType Directory
}

$Reports = Join-Path -Path $ReportPath -ChildPath $path

# Get MFA Status
Get-MFAStatusUsers | Sort-Object Name | Export-CSV -Path $Reports -NoTypeInformation

# Prepare Excel report path
$ExcelReportPath = $Reports -replace "\.csv$", ".xlsx"

# Export the data to Excel with formatting
$mfaStatusData | Export-Excel -Path $ExcelReportPath -WorksheetName "Data" -ClearSheet -AutoSize -AutoNameRange -TableStyle Medium9 -FreezeTopRow -BoldTopRow


# Calculate MFA Status Counts for Pie Chart
$mfaEnabledCount = ($mfaStatusData | Where-Object { $_."MFA_Status" -eq "enabled" }).Count

$mfaDisabledCount = ($mfaStatusData | Where-Object { $_."MFA_Status" -eq "disabled" }).Count

# Prepare Data for Pie Chart
$chartData = @(
    [PSCustomObject]@{Status = "MFA Enabled"; Count = $mfaEnabledCount},
    [PSCustomObject]@{Status = "MFA Disabled"; Count = $mfaDisabledCount}
)

# Add a new worksheet for the chart
$chartSheetName = "MFA Status Chart"
$chartData | Export-Excel -Path $ExcelReportPath -WorksheetName $chartSheetName -ClearSheet -AutoSize -StartRow 1 -StartColumn 1 -AutoNameRange -TableStyle Medium9 -BoldTopRow

# Add pie chart to the Chart worksheet
$excelPackage = Open-ExcelPackage -Path $ExcelReportPath

$worksheet = $excelPackage.Workbook.Worksheets[$chartSheetName]

# Adding data to the worksheet for the chart
$row = 2
foreach ($item in $chartData) {
    $worksheet.Cells["A$row"].Value = $item.Status
    $worksheet.Cells["B$row"].Value = $item.Count
    $row++
}

# Define the range for the chart
$labelsRange = "A2:A3"
$valuesRange = "B2:B3"

# Add the chart
if ($item.count -gt 0){
Add-ExcelChart -Worksheet $worksheet -ChartType PieExploded3D -LegendPosition Right -XRange $labelsRange -YRange $valuesRange -Title "MFA Status Distribution"
}

Close-ExcelPackage $excelPackage -Show

# Checks if file was written to path
if ((Get-Item $Reports).Length -gt 0) {
  Write-Host "Report finished and saved in $ReportPath" -ForegroundColor Green
  Write-Host "Disconnecting from Microsoft Graph" -ForegroundColor Cyan
  Disconnect-MgGraph
}else{
  Write-Host "Failed to create report" -ForegroundColor Red
  Write-Host "Disconnecting from Microsoft Graph" -ForegroundColor Cyan
  Disconnect-MgGraph
}
```
{: file='Get-MFAStatus.ps1'}
