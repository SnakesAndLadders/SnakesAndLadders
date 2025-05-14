---
title: PowerShell Daily Drivers
updated: 2025-05-14 23:09:55Z
created: 2025-05-14 22:53:07Z
---

# PowerShell Commands I Have Found Handy

I use PowerShell almost daily for my clients. I support mainly Windows clients and can open a remote shell without having to take over their desktop, so I have made an effort to collect handy commands that I can use frequently.

* * *

## [Winget](https://winget.run/)

Winget is probably the handiest way to install software on a remote system

Here is a quick example. Normally you can simply run   ` winget install [package] `   but when you are logging in as a system account you may have to use this one-liner instead.

```powershell
$install = "Adobe.Acrobat.Reader.64-bit"; $wingetdir = (Resolve-Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe" | Sort-Object -Property Path | Select-Object -Last 1); cd $wingetdir; .\winget.exe install -e --id $install --silent --accept-package-agreements --accept-source-agreements --scope machine
```

## Winget Script Install List

You can also automate the installation of multiple packages with a script like this

```powershell
#Install New apps
    $apps = @(
        @{name = "Microsoft.AzureCLI" }, 
        @{name = "Microsoft.PowerShell" }, 
        @{name = "Microsoft.VisualStudioCode" }, 
        @{name = "Microsoft.WindowsTerminal"; source = "msstore" }, 
        @{name = "Microsoft.PowerToys" }, 
        @{name = "Git.Git" }, 
        @{name = "Docker.DockerDesktop" },
        @{name = "Microsoft.dotnet" },
        @{name = "Notepad++.Notepad++" },
        @{name = "Joplin.Joplin" },
        @{name = "VideoLAN.VLC" },
        @{name = "AgileBits.1Password" },
        @{name = "Doist.Todoist" },
        @{name = "ZeroTier.ZeroTierOne" },
        @{name = "magic-wormhole.magic-wormhole" },
        @{name = "Canonical.Ubuntu.2204" },
        @{name = "GNU.nano" },
        @{name = "GitHub.cli" }
    );
    Foreach ($app in $apps) {
        #check if the app is already installed
        $listApp = winget list --exact -q $app.name
        if (![String]::Join("", $listApp).Contains($app.name)) {
            Write-host "Installing:" $app.name
            if ($app.source -ne $null) {
                winget install --exact --silent $app.name --source $app.source --accept-package-agreements
            }
            else {
                winget install --exact --silent $app.name --accept-package-agreements 
            }
        }
        else {
            Write-host "Skipping Install of " $app.name
        }
    }
```

Run the script, or you can add it as a gist and run it remotely! Feel free to use this one if you like, or copy my gist and make it your own.

```
 PowerShell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://gist.githubusercontent.com/SnakesAndLadders/07569d4760d2857dfdc8fba6e032b382/raw'))"
```

* * *

## Uninstall Software

If you would like to remove software that you did not install with Winget you can try this method

```powershell
Get-WmiObject -Class Win32_Product | Select-Object -Property Name
$ToRemove = "[name of software]"
$MyApp = Get-WmiObject -Class Win32_Product | Where-Object{$_.Name -eq $ToRemove}
$MyApp.Uninstall()
```

* * *

## User Management

```PowerShell
# User Management 
get-localuser 
disable-localuser <name>
query user /server:$SERVER #Show who is currently logged on
Logoff <sessionid>

# Create Local User
$password = Read-Host -AsSecureString
New-LocalUser -Name "LazyUser" -Password $password -FullName "Lazy User" -Description "Test user"

#Change a Password
$Password = Read-Host "Enter the new password" -AsSecureString
$UserAccount = Get-LocalUser -Name "admin"
$UserAccount | Set-LocalUser -Password $Password
```

## Create a Windows Service

```powershell
$params = @{
  Name = "TestService"
  BinaryPathName = 'C:\Path\To\file.exe'
  DependsOn = "NetLogon"
  DisplayName = "Test Service"
  StartupType = "Automatic"
  Description = "This is a test service."
}
New-Service @params

# View the service
Get-CimInstance -ClassName Win32_Service -Filter "Name='testservice'"
```

* * *

## Misc

Just some quick one liners

```powershell
rename-computer # run this by itself and it wiill prompt for new name

# Get last bootup time
(gcim cim_operatingsystem -comp $computername).LastBootUpTime

# Fix Windows boot entries (This is a CMD command but it works in PowerShell as well)
bcdedit # get the list
bcdedit /delete {identifier} # delete anything that is not {bootmgr} or c:\Windows\system32\winload.efi

# Process Management
get-process testservice
stop-process testservice

# Stop computer from timing out/going into standby or hibernate
powercfg.exe -x -monitor-timeout-ac 0
powercfg.exe -x -disk-timeout-ac 0
powercfg.exe -x -standby-timeout-ac 0
powercfg.exe -x -hibernate-timeout-ac 0
```

* * *

## Generate Passwords

I add these to my PowerShell profile so I can use them whenever I need

```PowerShell
# Create an 8 digit random PIN
function genpin {
    $digits = 0..9; 
    $randomDigits = $digits | Get-Random -Count 8; 
    $randomNumber = -join $randomDigits | Set-Clipboard;Write-Host "Copied to clipboard" -ForegroundColor Green
}

# Generate 16 character random secure password
function genpass {
    Add-Type -AssemblyName System.Web; 
    [System.Web.Security.Membership]::GeneratePassword(16, 4) | Set-Clipboard;Write-Host "Copied to clipboard" -ForegroundColor Green
}

# Generate Office 365 style password 
function gen365 {
    $randomNumber = Get-Random -Minimum 65 -Maximum 91
    $randomCapitalLetter = [char]$randomNumber
    $lowercaseLetters = -join ((65..90 | ForEach-Object { [char]$_ }) | Get-Random -Count 3)
    $lowercaseLetters = $lowercaseLetters.ToLower()
    $numbers = -join (0..9 | Get-Random -Count 5)
    $password = "$randomCapitalLetter$lowercaseLetters$numbers"  
    $password | Set-Clipboard;Write-Host "Copied to clipboard" -ForegroundColor Green
}

# Generate a "Correct_Horse_Battery_Staple" style password
Function genwords {
    Param
    (
        [Parameter(Mandatory=$false)]
        [INT]$NumberofWords = 3
    )
    $endpoint = "https://random-word-api.vercel.app/api?words=1"
    $arPW = @()
    $symbols = '~', '!', '@', '#', '$', '%', '^', '&', '*', '(', ')', '_'
    For ($i=0; $i -le $NumberofWords - 1; $i++) {

        Write-Progress `
            -Activity "Generating Password..."`
            -Status "$i of $NumberofWords"`
            -PercentComplete (($i / $NumberofWords)*100)

        $thisword = Invoke-RestMethod -Uri $endpoint
        $arPW += "$($thisword.Substring(0,1).toupper())$($thisword.Substring(1))"
        $arPW += $symbols | Get-Random
    }
    $PW = ($arPW) -join ""
    $PW = $PW.Substring(0,$PW.Length-1)
    $PW | Set-Clipboard;Write-Host "Copied to clipboard" -ForegroundColor Green
    Return $PW
}
```

* * *

## Show Folder Sizes

Run this from any directory to list the size of each folder

```bash
$fso = new-object -com Scripting.FileSystemObject; gci -Directory | select @{l='Size'; e={$fso.GetFolder($_.FullName).Size}},FullName | sort Size -Descending | ft @{l='Size [MB]'; e={'{0:N2}    ' -f ($_.Size / 1MB)}},FullName
```

* * *

&nbsp;