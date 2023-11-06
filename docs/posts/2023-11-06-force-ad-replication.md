---
draft: false
comments: true
date: 2023-11-06
categories:
  - Powershell
  - Scripting
  - Active Directory
---


# Force AD Replication
The below script will get all domain controllers in the current domain and then run a repadmin /syncall on all of them.

```powershell
(Get-ADDomainController -Filter *).Name | Foreach-Object {
    repadmin /syncall $_ (Get-ADDomain).DistinguishedName /e /A | Out-Null
}
```