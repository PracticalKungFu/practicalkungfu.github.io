---
comments: true
draft: false
date: 2023-12-06
categories:
  - Powershell
  - Scripting
  - Minecraft
authors:
  - tupcakes
---


# Automating Mod Downloads from Curseforge for Servers

I like modded Mincraft. I was trying to setup a modded server from Curseforge and found out that there are a number of mods that are flagged to not auto download with the rest of the pack. During the server pack install a file called "MODS_NEED_DOWNLOAD.txt" will be created containing various mods that meed this criteria. The expectation being you need to download each one and copy it to the server pack mods directory.

<!-- more -->

I wanted to streamline it a little bit. Normally for non-curseforge desktop clients, they will open the download links for all the mods at once. Its not great, but not terrible for getting the mods. At the very least, I figured this could be duplicated in powershell.

Sadly "MODS_NEED_DOWNLOAD.txt" is not a CSV file or anything else that is a useful structure, so we need to massage the data a little bit.

- We remove the first two lines from the file since we don't need them.
- Loop through all lines in the file.
  - Get the url from the current line.
  - Change the url to be the download link for the file. in question.
  - Open the download link.

## Script
```powershell
$file = 'C:\path_to_mc_server\MODS_NEED_DOWNLOAD.txt'
# Skip the first two lines of the file
$lines = (Get-Content $file | Select-Object -Skip 2)
# parse each line
foreach ($line in $lines) {
    # get the url from the line
    $url = "https" + ($line -split 'https')[1] -replace " ",""
    # change the url to be the download link
    $url = $url -replace "files","download"

    # start the download in your default browser
    Start-Process $url
}
```

The mods can now be easily copied to the mods folder of the server.

## Wrappup
Originally I tried using Invoke-WebRequest and BITS transfers, however I couldn't figure out a way to make them deal with the wait timer the download pages have.

I also looked at just downloading directly from their CDN, which I suspect I could make work with enough fiddling. The CDN url appears to just be in the format of:

```
https://mediafilez.forgecdn.net/files/4900/889/extra_compat-1.3.5.jar
```

Where "4900/889" appears to be just a split up string of the file ID for the mod. However it just wasn't worth it for how often I would need to run this. In the end it was just easier to convert to the download url and open a browser with the url.