---
comments: true
date: 2023-11-07
categories:
  - Windows
  - Server
  - DNS
---


# MS DNS Logging
For auditing we have to setup our DNS servers to audit various events and queries. 

## Set logging settings
The below command can be run on one of the dns servers. It will enable logging to a file at c:\dnslogs and enable log rollover.
```powershell
$setupFolder = "c:\dnslogs"
New-Item -Path $setupFolder -type directory -Force
Set-DnsServerDiagnostics `
-SaveLogsToPersistentStorage:$False `
-Queries:$True `
-Answers:$True `
-Notifications:$False `
-Update:$False `
-QuestionTransactions:$True `
-UnmatchedResponse:$True `
-SendPackets:$True `
-ReceivePackets:$True `
-TcpPackets:$True `
-UdpPackets:$True `
-FullPackets:$False `
-EventLogLevel 7 `
-UseSystemEventLog:$False `
-EnableLoggingToFile:$True `
-EnableLogFileRollover:$True `
-LogFilePath C:\dnslogs\dns.log `
-MaxMBFileSize 500000000 `
-WriteThrough:$False `
-EnableLoggingForLocalLookupEvent:$True `
-EnableLoggingForPluginDllEvent:$True `
-EnableLoggingForRecursiveLookupEvent:$True `
-EnableLoggingForRemoteServerEvent:$True `
-EnableLoggingForServerStartStopEvent:$True `
-EnableLoggingForTombstoneEvent:$True `
-EnableLoggingForZoneDataWriteEvent:$True `
-EnableLoggingForZoneLoadingEvent:$True
```

## Replicate logging settings
Change the DC name to match the one that you setup logging on. The command will get all the settings from that server and then copy them to the remaining DCs in the domain.
```powershell
$settings = Get-DnsServerDiagnostics -ComputerName DC01

$dcs = (Get-ADDomainController -Filter *).HostName

foreach ($dc in $dcs) {
    # create dnslogs folder
    Invoke-Command -ComputerName $dc -ScriptBlock { 
        $setupFolder = "c:\dnslogs"
        New-Item -Path $setupFolder -type directory -Force
    }
    
    $settings | Set-DnsServerDiagnostics -ComputerName $dc
}
```
