---
layout: post
title: Time To Transition To PowerShell Core For Real?
thumbnail: "assets/img/thumbnails/WinPSInCoreBanner.png"
tags: [Windows, PowerShell, PowerShell Core, Core, pwsh, Compatibility, ActiveDirectory]
excerpt_separator: <!--more-->
---

![Hansel Inside the Computer]({{ site.baseurl }}/assets/img/pexels/WinPSInCore.png){: .center-image }

I knew I wouldn't start using PowerShell Core (Pwsh) as my primary shell on Windows unless I had an easy way to use Cmdlets/Scripts/Functions/Modules that could only run on Windows PowerShell 5.1 (WinPS). So I decided to write a function that could push all compatible elements of my Pwsh environment to WinPS, and then run the WinPS exclusive operations. As luck would have it, a few hours after I pushed my *Get-WinPSInCore* function to GitHub, I discovered the *WindowsCompatibility Module* (version 0.0.1 was released by the PowerShell Team et al. a few days ago). So, I decided to kick the tires on the Module and do some (very limited) comparisons to my *Get-WinPSInCore* function.  
<!--more-->

# About Get-WinPSInCore and WindowsCompatibility Module

**Get-WinPSInCore Function On GitHub:**

[https://github.com/pldmgg/misc-powershell/blob/master/MyFunctions/PowerShellCore_Compatible/Get-WinPSInCore.ps1](https://github.com/pldmgg/misc-powershell/blob/master/MyFunctions/PowerShellCore_Compatible/Get-WinPSInCore.ps1)

**WindowsCompatibility Module On GitHub:**

[https://github.com/PowerShell/WindowsCompatibility](https://github.com/PowerShell/WindowsCompatibility)

**WindowsCompatibility Module On PSGallery:**

[https://www.powershellgallery.com/packages/WindowsCompatibility](https://www.powershellgallery.com/packages/WindowsCompatibility)

At a glance, the main difference between the *Get-WinPSInCore* function and the *WindowsCompatibility Module* is as follows:

The *WindowsCompatibility Module* makes it seem like WinPS Cmdlets / Scripts / Functions / Modules are loaded directly in Pwsh whereas my *Get-WinPSInCore* function makes it seem like you're running WinPS Cmdlets / Scripts / Functions / Modules in WinPS. At the end of the day, both approaches open a PSSession to Windows PowerShell 5.1, run the WinPS operations in WinPS, and bring the results back to Pwsh. (Of course, depending on the type of resulting WinPS object, some info might be lost in translation/serialization).

In the below examples, I'll try and demo both methods alongside each other.

# Demo

Before trying the following examples, make sure you have the latest version of Pwsh Installed. You can use my *Update-PowerShellCore* function to take care of this. (NOTE: The following will work both in Windows PowerShell 5.1 and an older version of PowerShell Core.)

```powershell
$pldmggFunctionsUrl = "https://raw.githubusercontent.com/pldmgg/misc-powershell/master/MyFunctions"

if (![bool]$(Get-Command Update-PowerShellCore -ErrorAction SilentlyContinue)) {
    $UpdatePSCoreFunctionUrl = "$pldmggFunctionsUrl/PowerShellCore_Compatible/Update-PowerShellCore.ps1"
    try {
        Invoke-Expression $([System.Net.WebClient]::new().DownloadString($UpdatePSCoreFunctionUrl))
    }
    catch {
        Write-Error $_
        Write-Error "Unable to load the Update-PowerShellCore function! Halting!"
        $global:FunctionResult = "1"
        return
    }
}

Update-PowerShellCore -DownloadDirectory "$HOME\Downloads" -Latest
```

Next, launch PowerShell Core (pwsh.exe), load the *Get-WinPSInCore* function, and install/import the *WindowsCompatibility Module*:

```powershell
$pldmggFunctionsUrl = "https://raw.githubusercontent.com/pldmgg/misc-powershell/master/MyFunctions"

if (![bool]$(Get-Command Get-WinPSInCore -ErrorAction SilentlyContinue)) {
    $GetWinPSInCoreFunctionUrl = "$pldmggFunctionsUrl/PowerShellCore_Compatible/Get-WinPSInCore.ps1"
    try {
        Invoke-Expression $([System.Net.WebClient]::new().DownloadString($GetWinPSInCoreFunctionUrl))
    }
    catch {
        Write-Error $_
        Write-Error "Unable to load the Get-WinPSInCore function! Halting!"
        $global:FunctionResult = "1"
        return
    }
}

Install-Module WindowsCompatibility
Import-Module WindowsCompatibility
```

Now we're ready to try out some examples!

# Example 1: Use the Windows PowerShell Module 'ActiveDirectory' Within PowerShell Core

Using the *Get-WinPSInCore* function...

```powershell
# NOTE: The Get-WinPSInCore function has an alias called 'shim'
PS C:\Users\zeroadmin> shim {
>> Import-Module ActiveDirectory
>> $(Get-ADComputer -Filter *)[0]
>> }

RunspaceId        : 286a6a47-a02d-4355-ab24-2064a59f81e2
DistinguishedName : CN=ZERODC01,OU=Domain Controllers,DC=zero,DC=lab
DNSHostName       : ZeroDC01.zero.lab
Enabled           : True
Name              : ZERODC01
ObjectClass       : computer
ObjectGUID        : e3616dd0-80e6-4c9c-b104-7b3e13fb1274
SamAccountName    : ZERODC01$
SID               : S-1-5-21-1048399111-1379594113-1692093488-1002
UserPrincipalName :
```

Using the *WindowsCompatibility Module*...

```powershell
PS C:\Users\zeroadmin> Import-WinModule ActiveDirectory
PS C:\Users\zeroadmin> $(Get-ADComputer -Filter *)[0]

RunspaceId        : ff6ad512-67b8-4b64-b215-6e982539d640
DistinguishedName : CN=ZERODC01,OU=Domain Controllers,DC=zero,DC=lab
DNSHostName       : ZeroDC01.zero.lab
Enabled           : True
Name              : ZERODC01
ObjectClass       : computer
ObjectGUID        : e3616dd0-80e6-4c9c-b104-7b3e13fb1274
SamAccountName    : ZERODC01$
SID               : S-1-5-21-1048399111-1379594113-1692093488-1002
UserPrincipalName :
```

SIDE NOTE: I know the PowerShell Team is working hard on making a version of the ActiveDirectory Module that works natively with PowerShell Core. Thank you!

# Example 2: Forward All PowerShell Core Environment Characteristics to Windows PowerShell and Use Them

First, let's set a few variables and functions in our PowerShell Core Session so that we have stuff to work with.

```powershell
PS C:\Users\zeroadmin> $HttpClient = [System.Net.Http.HttpClient]::new()
PS C:\Users\zeroadmin> $WebGetResult = $HttpClient.GetAsync("https://mathieubuisson.github.io/pester-custom-assertions")
PS C:\Users\zeroadmin> function Test-Func {'Output from Test-Func!!'}
PS C:\Users\zeroadmin> $OtherVar = "OtherStuff"
```

Using the *Get-WinPSInCore* function...

```powershell
PS C:\Users\zeroadmin> shim {
>> Test-Func
>> $WebGetResult
>> $OtherVar
>> } -VariablesToForward 'WebGetResult','OtherVar' -FunctionsToForward 'Test-Func'
Output from Test-Func!!

Result                  : @{Version=; Content=; StatusCode=200; ReasonPhrase=OK; Headers=System.Collections.ArrayList; RequestMessage=;
                          IsSuccessStatusCode=True}
Id                      : 2
Exception               :
Status                  : 5
IsCanceled              : False
IsCompleted             : True
IsCompletedSuccessfully : True
CreationOptions         : 0
AsyncState              :
IsFaulted               : False
AsyncWaitHandle         : @{Handle=; SafeWaitHandle=}
CompletedSynchronously  : False
RunspaceId              : 286a6a47-a02d-4355-ab24-2064a59f81e2

OtherStuff
```

SIDE NOTE: If you ever need/want to forward all Pwsh environment characteristics to WinPS using the *Get-WinPSInCore* function, you can do something like:

```powershell
shim {'Hello'} -VariablesToForward * -EnvironmentVariablesToForward * -FunctionsToForward * -ModulesToForward *
```

Using the *WindowsCompatibility Module*...

```powershell
PS C:\Users\zeroadmin> $WinCompatPSSession = Get-PSSession -Name $("win-" + $($(whoami) -split '\\')[-1])
PS C:\Users\zeroadmin> Invoke-Command -Session $WinCompatPSSession -ScriptBlock {
>> Invoke-Expression $args[0]
>> Test-Func
>> $args[1]
>> $args[2]
>> } -ArgumentList $(${Function:Test-Func}.Ast.Extent.Text),$WebGetResult,$OtherVar -HideComputerName
Output from Test-Func!!

RunspaceId              : ff6ad512-67b8-4b64-b215-6e982539d640
Result                  : StatusCode: 200, ReasonPhrase: 'OK', Version: 1.1, Content: System.Net.Http.NoWriteNoSeekStreamContent, Headers:
                          {
                            Cache-Control: max-age=600
                            Connection: keep-alive
                            Date: Sun, 10 Jun 2018 13:36:35 GMT
                            Via: 1.1 varnish
                            Accept-Ranges: bytes
                            Age: 107
                            Server: GitHub.com
                            Vary: Accept-Encoding
                            Access-Control-Allow-Origin: *
                            X-GitHub-Request-Id: ACA2:55A1:65855C:89F8B7:5B1D25B6
                            X-Served-By: cache-dca17741-DCA
                            X-Cache: HIT
                            X-Cache-Hits: 1
                            X-Timer: S1528637795.226039,VS0,VE1
                            X-Fastly-Request-ID: 4bcfe019dd345cd730266486b7dc923614eaf3cd
                            Content-Length: 63360
                            Content-Type: text/html; charset=utf-8
                            Expires: Sun, 10 Jun 2018 13:30:55 GMT
                            Last-Modified: Sat, 09 Jun 2018 15:24:39 GMT
                          }
Id                      : 2
Exception               :
Status                  : RanToCompletion
IsCanceled              : False
IsCompleted             : True
IsCompletedSuccessfully : True
CreationOptions         : None
AsyncState              :
IsFaulted               : False
AsyncWaitHandle         : System.Threading.ManualResetEvent
CompletedSynchronously  : False

OtherStuff
```

# My Thoughts

I haven't fully explored the *WindowsCompatibility Module* yet, but given my testing thus far, I'm pretty comfortable saying the following:

* The *WindowsCompatibility Module* will become (and should become) the most popular method of running Windows PowerShell 5.1 in PowerShell Core
* At this point in time, if you want to [try to] use Pwsh variables/functions/Modules in WinPS, use the *Get-WinPSInCore* function.
* If you want to use WinPS cmdlets/functions/Modules in Pwsh, use the *WindowsCompatibility Module*. 

# Next Post Teaser

I know in my [previous post](https://pldmgg.github.io/2018/06/02/MiniLab.html) I said that this week I was going to write about standing up PKI using CloudFlare's CFSSL and Docker Containers...but when I started down that road, this is the post I ended up with :/ I'll try for next week!