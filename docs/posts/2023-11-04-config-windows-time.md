---
comments: true
date: 2023-11-04
categories:
  - Windows
  - ntp
  - aws
---


# Configure Windows Time
## AD members
This will reset a domain member to use NT5DS to set the time. This should be the default for "most" domain environments
```
net stop w32time
w32tm /unregister
w32tm /register
net start w32time
w32tm /config /syncfromflags:DOMHIER /update
net stop w32time
net start w32time
w32tm /resync /rediscover /nowait
```

## AD PDC
In most setups the PDC should be configured to sync with an external time source. Other domain controllers should sync to it. The below command simply sets the PDC to sync with the NIST time source.
```
net stop w32time
w32tm /unregister
w32tm /register
net start w32time
w32tm.exe /config /manualpeerlist:"time.nist.gov,0x8" /syncfromflags:manual /reliable:YES /update
net stop w32time
net start w32time
w32tm /resync /nowait
```

## Set timezone
```
tzutil /s "Central Standard Time"
```

## Troubleshooting
```
w32tm /resync /rediscover
w32tm /query /status /verbose
w32tm /query /configuration
```

## AWS Workspace Default
Below is the new time settings for AWS machines. This command is if you want to revert to the default setting for AWS.
```
w32tm /config /manualpeerlist:"169.254.169.123,0x9 time.windows.com,0x8" /syncfromflags:manual /update
```