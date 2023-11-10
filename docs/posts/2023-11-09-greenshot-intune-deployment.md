---
comments: true
date: 2023-11-09
categories:
  - Intune
  - Powershell
authors:
  - tupcakes
---


# Greenshot Intune Deployment

Greenshot is a very nice program. However there is one thing that REALLY annoys me about it. The hard coded browser poppup after install. Deploying Greenshot via Intune is not the most straight foward thing if you want to customize the install. Plus there is having to kill greenshot running under the deploy account (usually system) and also starting greenshot under the current logged on user after install.

For this I used [this intune winget template](https://github.com/FlorianSLZ/scloud/blob/main/winget/winget-program-template/winget-program-template.zip). It consists of your usual detection, install, and uninstall scripts. Below I'll be going over the install script.



```powershell
Param
    (
    [parameter(Mandatory=$false)]
    [String[]]
    $param
    )


#Attempt to stop greenshot if not kill it with fire
if (Test-Path "$ENV:ProgramFiles\Greenshot\Greenshot.exe") {
    Start-Process -FilePath "$ENV:ProgramFiles\Greenshot\Greenshot.exe /exit"
} else {}
Start-Sleep -Seconds 10
Stop-Process -Name greenshot -Force -Confirm:$false -ErrorAction SilentlyContinue


#create custom installer settings and save to temp location.
$custominf = @'
[Setup]
Lang=en
Group=Greenshot
NoIcons=0
SetupType=custom
Components=greenshot,plugins\office,plugins\ocr,plugins\externalcommand
Tasks=startup
'@
New-Item C:\temp\custom.inf -Force
Set-Content C:\temp\custom.inf $custominf


# do the actual install of greenshot using winget and the custom settings we saved in the last step
$ProgramName = "Greenshot.Greenshot"
$Path_local = "$Env:Programfiles\_MEM"
Start-Transcript -Path "$Path_local\Log\$ProgramName-install.log" -Force -Append

# resolve winget_exe
$winget_exe = Resolve-Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe\winget.exe"
if ($winget_exe.count -gt 1){
        $winget_exe = $winget_exe[-1].Path
}

if (!$winget_exe){Write-Error "Winget not installed"}

& $winget_exe install --exact --id $ProgramName --silent --accept-package-agreements --accept-source-agreements --scope=machine $param --override "/LOADINF=C:\temp\custom.inf /NORESTART /VERYSILENT"


#create a fixted settings file and copy it to the greenshot install directory. These settings are enforced in the app.
$fixedsettings = @'
[Core]
; The language in IETF format (e.g. en-US)
Language=en-US
; Comma separated list of Plugins which are allowed. If something in the list, than every plugin not in the list will not be loaded!
IncludePlugins=External command Plugin,OCR Plugin,Office Plugin
; Comma separated list of Plugins which are NOT allowed.
ExcludePlugins=
; Comma separated list of destinations which should be disabled.
ExcludeDestinations=Box,Confluence,Dropbox,Flickr,Imgur,Jira,Photobucket,Picasa,
; How many days between every update check? (0=no checks)
; UpdateCheckInterval=0
'@

if (-Not (Test-Path "$ENV:ProgramFiles\Greenshot\Greenshot-fixed.ini")) {
    New-Item "$ENV:ProgramFiles\Greenshot\Greenshot-fixed.ini"
    Set-Content "$ENV:ProgramFiles\Greenshot\Greenshot-fixed.ini" $fixedsettings
}


#here is the method I've been using to get around the forced browser poppup.
#stop deploy user browser instances and greenshot
Get-Process iexplore -IncludeUserName -ErrorAction SilentlyContinue | Where-Object UserName -match $env:username | Stop-Process -ErrorAction SilentlyContinue
Get-Process chrome -IncludeUserName -ErrorAction SilentlyContinue | Where-Object UserName -match $env:username | Stop-Process -ErrorAction SilentlyContinue
Get-Process msedge -IncludeUserName -ErrorAction SilentlyContinue | Where-Object UserName -match $env:username | Stop-Process -ErrorAction SilentlyContinue
Get-Process greenshot -IncludeUserName -ErrorAction SilentlyContinue | Where-Object UserName -match $env:username | Stop-Process -ErrorAction SilentlyContinue


#start greenshot as logged on user
#temporarily download psexec to temp location.
#run greenshot via the console session (vdi might require another solution)
if (Test-Path "$ENV:ProgramFiles\Greenshot\Greenshot.exe") {
    $source = 'https://live.sysinternals.com/PsExec.exe'
    $destination = 'C:\temp\PsExec.exe'
    Invoke-WebRequest -Uri $source -OutFile $destination
    .\PsExec.exe -accepteula -i -d "$ENV:ProgramFiles\Greenshot\greenshot.exe"
    Remove-Item C:\temp\PsExec.exe -Confirm:$false
} else {}


#cleaup
if (Test-Path "C:\temp\custom.inf") {
    Remove-Item "C:\temp\custom.inf" -Confirm:$false
}

Stop-Transcript
```