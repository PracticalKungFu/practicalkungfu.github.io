---
comments: true
date: 2023-11-10
categories:
  - Powershell
  - Scripting
authors:
  - tupcakes
---


# Poor Mans Connection Monitoring

This morning I had need to be alerted if I ping starts failing in an ad-hoc situation. The below script does a test ping and then beeps if the ping fails. It'll loop endlessly.


```powershell 
while ($true) {
    $ping = Test-Connection -ComputerName 10.120.10.100 -Count 1
    $ping
    Start-Sleep -Seconds 1
    If (($ping -eq "") -or ($ping -eq $null)) {
        [console]::beep(500,1000)
    }
}
```