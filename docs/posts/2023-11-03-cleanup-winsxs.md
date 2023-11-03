---
comments: true
date: 2023-11-03
categories:
  - Windows
---


# Cleanup WinSXS
From: [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/clean-up-the-winsxs-folder?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/clean-up-the-winsxs-folder?view=windows-11)

## Analyze component store
```
dism /Online /Cleanup-Image /AnalyzeComponentStore
```
## Remove previous versions of updated components
```
Dism.exe /online /Cleanup-Image /StartComponentCleanup
```

## Remove all superseded versions of components
```
Dism.exe /online /Cleanup-Image /StartComponentCleanup /ResetBase
```