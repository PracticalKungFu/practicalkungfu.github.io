---
comments: true
draft: false
date: 2023-12-07
categories:
  - DNS
authors:
  - tupcakes
---


# Exporting and Importing MS DNS Zones

Yes it's possible via the command line. Learn from my mistake. When you import the zone using the /primary option, it isn't AD integrated. And the /dsprimary option doesn't take a file parameter. :(

<!-- more -->

## Export
Export the zone from the source server.
```
dnscmd /zoneexport "domain.com" "domain.com.zone"
```

## Import
Import the zone on the new server.
```
dnscmd vm-dc01 /zoneadd "domain.com" /primary /file "domain.com.zone" /load
```

!!! tip "Make sure to change the zone to be AD Integrated"

    The /dsprimary flag used to create an AD integrated zone can't import a file for some reason. Just import as a regular zone and then change to AD integrated via the gui.