---
layout: post
title: Refactoring Windows PowerShell 5.1 for PowerShell Core 6.X
thumbnail: "assets/img/thumbnails/PSCompat.png"
tags: [Windows, PowerShell, PowerShell Core, Core, pwsh, Compatibility, ActiveDirectory]
excerpt_separator: <!--more-->
---

![Hansel Inside the Computer]({{ site.baseurl }}/assets/img/pexels/PSCompat.png){: .center-image }

Last week I finally decided to rollup my sleeves and attempt to refactor some of my more recent Windows PowerShell 5.1 code to work with PowerShell Core 6.X (on Windows). My goal was to use the WindowsCompatibility Module as efficiently as possible so that I really didn't have to touch the majority of my existing code. The experience was relatively painless, but I still wanted to share some lessons learned as well as a way to make (most of) your existing code compatible with PowerShell Core by adding **only two lines** towards the beginning of your script/function/Module.
<!--more-->

**WindowsCompatibility Module On GitHub:**

[https://github.com/PowerShell/WindowsCompatibility](https://github.com/PowerShell/WindowsCompatibility)

**WindowsCompatibility Module On PSGallery:**

[https://www.powershellgallery.com/packages/WindowsCompatibility](https://www.powershellgallery.com/packages/WindowsCompatibility)

# Get to the (Bullet) Point(s)

First and foremost, I want reiterate that everything in this post applies to PowerShell Core **on Windows**, not Linux.

(SIDE NOTE: There is actually a relatively easy way of getting the WindowsCompatibility Module to work with PowerShell Core on Linux - effectively giving you access to Windows PowerShell 5.1 on Linux - but I suspect the PowerShell Team and contributors will be streamlining this in the future, so I'm not going to touch on it in this post).

That being said, here are some lessons learned:

- Install and import *all* required Modules *explicitly* at the beginning of your script/function/Module - *including* Modules that PowerShell normally loads for you automatically when you use an associated cmdlet.
- After you import a Module using the WindowsCompatibility Module's `Import-WinModule` cmdlet, make sure all of the commands you expect to be available are, in fact, available
- Pay extra attention to your use of type accelerators. Sometimes the underlying class doesn't exist in PowerShell Core...sometimes the type accelerator itself just isn't set by default like it is in Windows PowerShell 5.1.
- Be very careful when using objects that come from Windows PowerShell 5.1 via the WindowsCapability Module's
implicit remoting. Any property that is (itself) an object will not be represented as expected (exceptions being string, bool, and int, which *are* expressed as expected). This is due to serialization/deserialization over the implicit remoting session.
- Be very careful when using `Add-Type`. Sometimes your C# can compile in PowerShell Core, sometimes it can't.
- Whenever you use `Invoke-WinCommand`, make sure you always use the `-ComputerName` parameter even if
it is just `localhost`. There are some situations where `Invoke-WinCommand` complains about not having this
parameter set explicitly, eventhough it shouldn't be necessary. I would open an issue on GitHub, but I can't recreate
it consistently.
- Don't use `Start-Job` - Setting up an equivalent WindowsCompatibility environment within the separate process
is a bug factory...Use my New-Runspace function instead:

[https://www.reddit.com/r/PowerShell/comments/8qjhtj/new_function_newrunspace_a_faster_alternative_to/](https://www.reddit.com/r/PowerShell/comments/8qjhtj/new_function_newrunspace_a_faster_alternative_to/)

# Helper Functions to the Rescue

Having stepped on all of the aforementioned landmines, I thought it would be a good idea to create a way to quickly gut-check existing Windows PowerShell 5.1 code in PowerShell Core before moving forward with any refactors. So, I created three helper functions that will allow you to quickly make a reasonable attempt at running your existing Windows PowerShell 5.1 code in PowerShell Core with very little effort.

My three PSCompatibility Functions are: `GetModuleDependencies`, `InvokePSCompatibility`, and `InvokeModuleDependencies`.

(NOTE: I didn't use dashes in the function names because I generally use them as Private Functions, and I'm a big fan of being able to quickly differentiate between Public and Private functions by simply looking for a dash.)

The `GetModuleDependencies` function finds all cmdlets/functions in your script/function/Module and maps them to their associated locally-available Module. The `InvokePSCompatibility` function contains logic that determines which Modules to install/import and (of equal importance) which Modules **not** to install/import. The `InvokeModuleDependencies` function contains logic that determines how Modules should be installed/imported: if you're within PowerShell Core, `InvokePSCompatibility` will be used; if you're within Windows PowerShell 5.1, Modules will be installed/imported using traditional methods.

In the end, the only function that you'll need to actively use is `InvokeModuleDependencies`. Also, under most circumstances, you'll probably only need to add two lines of code towards the beginning of your existing script/function/Module in order to make it compatible with PowerShell Core.

# InvokeModuleDependencies Usage

Before we jump into the below examples, download `PSCompatibilityFunctions.ps1` from my GitHub repo:

[https://github.com/pldmgg/misc-powershell/blob/master/MyFunctions/PowerShellCore_Compatible/PSCompatibilityFunctions.ps1](https://github.com/pldmgg/misc-powershell/blob/master/MyFunctions/PowerShellCore_Compatible/PSCompatibilityFunctions.ps1)

Download using PowerShell (5.1 or Core 6.X) via:

```powershell
$DownloadUri = "https://raw.githubusercontent.com/pldmgg/misc-powershell/master/MyFunctions/PowerShellCore_Compatible/PSCompatibilityFunctions.ps1"
Invoke-WebRequest -Uri $DownloadUri -OutFile "$HOME\Downloads\PSCompatibilityFunctions.ps1"
```

# The First (Somewhat Contrived) Example

Launch the latest version of PowerShell Core and try the following...

```powershell
PS C:\Users\zeroadmin> Set-Content -Path "$HOME\Get-NetworkInfo.ps1" -Value @'
function Get-NetworkInfo {
    [CmdletBinding()]
    Param (
        [Parameter(Mandatory=$False)]
        [switch]$IPv4Only = $True
    )

    if ($IPv4Only) {
        Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
    else {
        Get-NetIPAddress | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
}
'@

PS C:\Users\zeroadmin> . "$HOME\Get-NetworkInfo.ps1"
PS C:\Users\zeroadmin> Get-NetworkInfo
Get-NetIPAddress : The term 'Get-NetIPAddress' is not recognized as the name of a cmdlet, function, script file, or operable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At V:\powershell\Testing\PSCompatTest.ps1:9 char:9
+         Get-NetIPAddress -AddressFamily IPv4
+         ~~~~~~~~~~~~~~~~
+ CategoryInfo          : ObjectNotFound: (Get-NetIPAddress:String) [], CommandNotFoundException
+ FullyQualifiedErrorId : CommandNotFoundException

```

As you can see, PowerShell Core doesn't recognize the Networking Cmdlets that are part of the (built-in) Windows PowerShell 5.1 NetTCPIP Module. I **could** refactor this function by...

- Adding a check for the PowerShell version
- Using if/then statements that determine whether to install/import the WindowsCompatibility Module
- Using the cmdlets `Import-WinModule` and/or `Invoke-WinCommand` as appropriate

...but this get's really annoying very quickly if you're going through a larger code base.

So instead, let's just add a couple of lines at the beginning of `Get-NetworkInfo.ps1` and call it a day...

```powershell
PS C:\Users\zeroadmin> Set-Content -Path "$HOME\Get-NetworkInfo.ps1" -Value @'
. "$HOME\Downloads\PSCompatibilityFunctions.ps1"
$ModuleDependenciesMap = InvokeModuleDependencies

function Get-NetworkInfo {
    [CmdletBinding()]
    Param (
        [Parameter(Mandatory=$False)]
        [switch]$IPv4Only = $True
    )

    if ($IPv4Only) {
        Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
    else {
        Get-NetIPAddress | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
}
'@

PS C:\Users\zeroadmin> . "$HOME\Get-NetworkInfo.ps1"
PS C:\Users\zeroadmin> Get-NetworkInfo

IPAddress         : 192.168.2.46
InterfaceIndex    : 2
InterfaceAlias    : Ethernet
AddressFamily     : 2
Type              : 1
PrefixLength      : 24
PrefixOrigin      : 3
SuffixOrigin      : 3
AddressState      : 4
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : 1
```

Hooray! The `Get-NetworkInfo` function is now compatible with both Windows PowerShell 5.1 AND PowerShell Core 6.X!

Even though the above example involves dot sourcing a .ps1 file that contains the `Get-NetworkInfo` function, it's important to note that you can also just use `InvokeModuleDependencies` directly within a function loaded in memory. In other words, we could simply do the following directly in PowerShell Core and get the same result:

```powershell
PS C:\Users\zeroadmin> function Get-NetworkInfo {
    [CmdletBinding()]
    Param (
        [Parameter(Mandatory=$False)]
        [switch]$IPv4Only = $True
    )

    . "$HOME\Downloads\PSCompatibilityFunctions.ps1"
    $ModuleDependenciesMap = InvokeModuleDependencies

    if ($IPv4Only) {
        Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
    else {
        Get-NetIPAddress | Where-Object {$_.InterfaceAlias -notmatch "Loopback"}
    }
}
PS C:\Users\zeroadmin> Get-NetworkInfo

IPAddress         : 192.168.2.46
InterfaceIndex    : 2
InterfaceAlias    : Ethernet
AddressFamily     : 2
Type              : 1
PrefixLength      : 24
PrefixOrigin      : 3
SuffixOrigin      : 3
AddressState      : 4
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : 1
```

# Using InvokeModuleDependencies Within in a Module

You can also use `InvokeModuleDependencies` within a Module's .psm1 file to make your entire Module compatible with PowerShell Core! In this situation, best practice would be to add `GetModuleDependencies`, `InvokePSCompatibility`, and `InvokeModuleDependencies` as Private functions within your Module and then do something like the following at towards the beginning of your .psm1 file...

```powershell
# Get public and private function definition files.
[array]$Public  = Get-ChildItem -Path "$PSScriptRoot\Public\*.ps1" -ErrorAction SilentlyContinue
[array]$Private = Get-ChildItem -Path "$PSScriptRoot\Private\*.ps1" -ErrorAction SilentlyContinue
$ThisModule = $(Get-Item $PSCommandPath).BaseName

# Dot source the Private functions
foreach ($import in $Private) {
    try {
        . $import.FullName
    }
    catch {
        Write-Error -Message "Failed to import function $($import.FullName): $_"
    }
}

if (Test-Path "$PSScriptRoot\module.requirements.psd1") {
    $ModuleManifestData = Import-PowerShellDataFile "$PSScriptRoot\module.requirements.psd1"
    $ModulesToInstallAndImport = $ModuleManifestData.Keys | Where-Object {$_ -ne "PSDependOptions"}
}

$InvModDepSplatParams = @{
    RequiredModules                     = $ModulesToInstallAndImport
    InstallModulesNotAvailableLocally   = $True
}
$ModuleDependenciesMap = InvokeModuleDependencies @InvModDepSplatParams

# Public Functions
...<Truncated>...
```

IMPORTANT NOTE: Unfortunately, depending on the number of Module dependencies in your custom Module, using `InvokeModuleDependencies` will significantly increase the amount of time it takes to import your custom Module.

# Additional Examples

If your script/function/Module requires any Modules that are NOT already installed locally, then you can use the `-RequiredModules` parameter. 

IMPORTANT NOTE: If you're not sure if a particular Module is already installed/available locally, add it to the `-RequiredModules` list and use the `-InstallModulesNotAvailableLocally` switch. Since `InvokeModuleDependencies` checks local availability before attempting to download/install external Modules, there's no detriment to adding it to the `-RequiredModules` string array.

```powershell
PS C:\Users\zeroadmin> function New-LabBuilderLab {
    Param (
        [Parameter(Mandatory=$True)]
        [string]$ConfigPath,

        [Parameter(Mandatory=$True)]
        [string]$LabPath,

        [Parameter(Mandatory=$True)]
        [string]$LabName
    )

    . "$HOME\Downloads\PSCompatibilityFunctions.ps1"
    $ModuleDependenciesMap = InvokeModuleDependencies -RequiredModules @("LabBuilder") -InstallModulesNotAvailableLocally

    if (! $(Test-Path $LabPath)) {
        $null = New-Item -ItemType Directory -Path $LabPath
    }

    New-Lab -ConfigPath $ConfigPath -LabPath $LabPath -Name $LabName
}

PS C:\Users\zeroadmin> New-LabBuilderLab -ConfigPath "$HOME\NewLab\LabConfig" -LabPath "$HOME\NewLab" -LabName Test

    Directory: C:\Users\zeroadmin\NewLab


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        6/26/2018   2:20 PM              1 ISOFiles
d-----        6/26/2018   2:20 PM              1 VHDFiles

RunspaceId       : d40d6471-7e0e-4a3c-9ac4-407e2cf3f091
xml              : version="1.0" encoding="utf-8"
labbuilderconfig : labbuilderconfig

```

# Gotchas

Unfortunately, not all Modules can be imported cleanly using the WindowsCompatibility Module's `Import-WinModule` cmdlet. Sometimes, you will have to refactor your code directly using the `Invoke-WinCommand` cmdlet.

```powershell
PS C:\Users\zeroadmin> function Get-Permissions {
    [CmdletBinding()]
    Param (
        [Parameter(Mandatory=$False)]
        [string]$Account = $(whoami)
    )

    . "$HOME\Downloads\PSCompatibilityFunctions.ps1"
    $ModuleDependenciesMap = InvokeModuleDependencies -RequiredModules @("NTFSSecurity")

    Get-NTFSAccess -Path $(Get-Location).Path -Account $Account
}

PS C:\Users\zeroadmin> Get-Permissions
WARNING: The following Modules were not able to be loaded via implicit remoting:
NTFSSecurity
WARNING: All code within 'Get-Permissions' that uses these Modules must be refactored like:
Invoke-WinCommand -ComputerName localhost -ScriptBlock {
    <existing code>
}
WARNING: 'Get-Permissions' will probably *not* work within PowerShell Core!

Get-NTFSAccess : The term 'Get-NTFSAccess' is not recognized as the name of a cmdlet, function, script file, or operable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At V:\powershell\Testing\Get-Permissions.ps1:11 char:5
+     Get-NTFSAccess -Path $(Get-Location).Path -Account $Account
+     ~~~~~~~~~~~~~~
+ CategoryInfo          : ObjectNotFound: (Get-NTFSAccess:String) [], CommandNotFoundException
+ FullyQualifiedErrorId : CommandNotFoundException
```

The above example can be refactored by simply doing...

```powershell
function Get-Permissions {
    [CmdletBinding()]
    Param (
        [Parameter(Mandatory=$False)]
        [string]$Path
    )

    . "$HOME\Downloads\PSCompatibilityFunctions.ps1"
    $ModuleDependenciesMap = InvokeModuleDependencies -RequiredModules @("NTFSSecurity") -InstallModulesNotAvailableLocally

    if (!$Path) {
        $Path = $(Get-Location).Path
    }

    Invoke-WinCommand -ComputerName localhost -ScriptBlock {
        Get-NTFSAccess -Path $args[0]
    } -ArgumentList $Path
}

PS C:\Users\zeroadmin> Get-Permissions -Path $(Get-Location).Path -WarningAction SilentlyContinue

AccountType        :
PSComputerName     : localhost
RunspaceId         : 7bbff6b4-ff3a-44f3-8dbe-175511f1138f
Name               : zeroadmin
FullName           : C:\Users\zeroadmin
InheritanceEnabled : False
InheritedFrom      :
...<Truncated>...

```

And there you have it! Hope this can help some folks! Let me know what you think on [Reddit](https://www.reddit.com/user/fourierswager) or on [Twitter](https://twitter.com/pldmgg).
