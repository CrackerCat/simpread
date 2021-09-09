> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.alkanesolutions.co.uk](https://www.alkanesolutions.co.uk/2020/06/03/use-powershell-to-find-an-advertised-shortcut-target/)

> Have you ever looked at the target of an .lnk shortcut and it appears to be greyed out/disabled? You ......

Have you ever looked at the target of an .lnk shortcut and it appears to be greyed out/disabled? Chances are it is a Windows Installer advertised shortcut, which is used as part of Windows Installer resiliency and self healing. Instead of it typically pointing to an executable, it invokes the Windows Installer subsystem to locate the shortcut target instead.

```
function Get-AdvertisedShortcut {

    param([string]$pathToLnk)

    $shortcutTarget = ""
    
    if ($pathToLnk -ne $null -and (test-path $pathToLnk)) {    
        $windowsInstaller = New-Object -ComObject WindowsInstaller.Installer
        $lnkTarget = $WindowsInstaller.GetType().InvokeMember("ShortcutTarget","GetProperty",$null,$windowsInstaller,$pathToLnk)
        $productCode = $lnkTarget.GetType().InvokeMember("StringData","GetProperty",$null,$lnkTarget,1)
        $componentCode = $lnkTarget.GetType().InvokeMember("StringData","GetProperty",$null,$lnkTarget,3)
        $shortcutTarget = $WindowsInstaller.GetType().InvokeMember("ComponentPath","GetProperty",$null,$WindowsInstaller,@($productCode,$componentCode))        
    }

    return $shortcutTarget
}

$pathToLnk = Get-AdvertisedShortcut "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\InstEd.lnk"

```