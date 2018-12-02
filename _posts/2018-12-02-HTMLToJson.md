---
layout: post
title: HTMLToJson PowerShell Module
#thumbnail: "assets/img/thumbnails/MiniLab-Thumbnail.png"
tags: [PowerShell, PowerShell Core, pwsh, Splash, ScrapingHub, dotnet, dotnet-script, openscraping]
excerpt_separator: <!--more-->
---

So you've finally found a site with a ton of great information, but it hasn't implemented a proper REST API? You could use existing PowerShell Web Cmdlets to try and parse things, but that will only get you so far. The HTMLToJson Module leverages [OpenScraping](https://github.com/Microsoft/openscraping-lib-csharp), [dotnet-script](https://github.com/filipw/dotnet-script), and ScrapingHub's [Splash Server](https://github.com/scrapinghub/splash) in order to faithfully render websites (**including javascript**), and return your desired Json output.
<!--more-->

# About the HTMLToJson Module

**Compatibility**

All functions in the HTMLToJson Module except `Install-Docker` and `Deploy-SplashServer` are compatible with Windows PowerShell 5.1 and PowerShell Core 6.X (Windows and Linux). The `Install-Docker` and `Deploy-SplashServer` functions work on PowerShell Core 6.X on Linux (specifically Ubuntu 18.04/16.04/14.04, Debian 9/8, CentOS/RHEL 7, OpenSUSE 42).

**HTMLToJson Module On GitHub:**

[https://github.com/pldmgg/HTMLToJson](https://github.com/pldmgg/HTMLToJson)

**HTMLToJson Module On PowerShell Gallery:**

[https://www.powershellgallery.com/packages/HTMLToJson](https://www.powershellgallery.com/packages/HTMLToJson)

# Initial Prep

In order to fully and faithfully render sites, the HTMLToJson Module relies on ScrapingHub's Splash Server. If you do not already have Splash deployed to your environment, ssh to a VM running your preferred compatible Linux distro, launch PowerShell Core (using `sudo`), and install the HTMLToJson Module -

```
sudo pwsh
Install-Module HTMLToJson
exit
```

Next, launch pwsh (without `sudo`), import the HTMLToJson Module, and install Docker (you will receive a sudo prompt unless you have password-less sudo configured on your system).

```powershell
pwsh

Import-Module HTMLToJson
Install-Docker
```

Finally, deploy ScrapingHub's Splash Server Docker Container -

```powershell
Deploy-SplashContainer
```

At this point, you can continue on the same Linux VM running your Splash Docker container, or you can hop back into your local workstation (Windows or Linux). Either way, the following steps will be the same.

Next, we need to install the .Net Core SDK as well as dotnet-script. These provide the `dotnet` and `dotnet-script` binaries -

```powershell
Install-DotNetSDK
Install-DotNetScript
```

# Parsing A Website Using XPath

Let's say we want to parse the site `http://dotnetapis.com`. First we need to decide what the structure of our Json output should look like. For this site, one such configuration could be as follows -

```powershell
$JsonXPathConfigString = @"
{
    "title": "//*/h1",
    "VisibleAPIs": {
        "_xpath": "//a[(@class=\"list-group-item\")]",
        "APIName": ".//h3",
        "APIVersion": ".//p//code//span[normalize-space()][2]",
        "APIDescription": ".//p[(@class=\"list-group-item-text\")]"
    }
}
"@
```

The above is a Json Configuration that uses XPath in order to specify which element(s) we would like returned (specifically, the text contained within those elements). The Json property `"_xpath": "//a[(@class='list-group-item')]",` indicates that we would like all elements that match this XPath pattern to be returned in our `VisibleAPIs` list.

**SIDE NOTE:** Don't know XPath? Just navigate to the site in Chrome, right-click the element you're interested in, click 'Inspect Element', right-click the actual html, and select Copy -> Copy XPath.


So, let's see what the end result looks like. (Please note that the first time you run `Get-SiteAsJson` will be slower than subsequent runs. Also, be sure to change the below `-SplashServerUri` value to suit your environment.)

```powershell
Get-SiteAsJson -Url 'http://dotnetapis.com/' -XPathJsonConfigString $JsonXPathConfigString -SplashServerUri "http://localhost:8050"

{
  "title": "DotNetApis (BETA)",
  "VisibleAPIs": [
    {
      "APIName": "NUnit",
      "APIVersion": "3.11.0",
      "APIDescription": "NUnit is a unit-testing framework for all .NET languages with a strong TDD focus."
    },
    {
      "APIName": "Json.NET",
      "APIVersion": "12.0.1",
      "APIDescription": "Json.NET is a popular high-performance JSON framework for .NET"
    },
    {
      "APIName": "EntityFramework",
      "APIVersion": "6.2.0",
      "APIDescription": "Entity Framework is Microsoft's recommended data access technology for new applications."
    },
    {
      "APIName": "MySql.Data",
      "APIVersion": "8.0.13",
      "APIDescription": "MySql.Data.MySqlClient .Net Core Class Library"
    },
    {
      "APIName": "NuGet.Core",
      "APIVersion": "2.14.0",
      "APIDescription": "NuGet.Core is the core framework assembly for NuGet that the rest of NuGet builds upon."
    }
  ]
}

```

# Other Notes

- Some sites have infinite vertical scrolling implemented. If you're trying to parse a site that uses this feature, add the `-HandleInfiniteScrolling` switch to `Get-SiteAsJson`
- If you need to take a series of actions on a site in order to reveal certain content, you can write a Lua Script that instructs the Splash Server to perform those actions before returning the rendered html to be parsed. For an example of such a script, see [https://github.com/pldmgg/HTMLToJson/blob/master/HTMLToJson/LuaScripts/InfiniteScrolling.lua](https://github.com/pldmgg/HTMLToJson/blob/master/HTMLToJson/LuaScripts/InfiniteScrolling.lua). (Even if you're not familiar with Lua, it's pretty straightforward.)

# Shoulders of Giants

Big thanks to [zmarty](https://github.com/zmarty) for OpenScraping, [filipw](https://github.com/filipw) and [seesharper](https://github.com/seesharper) for dotnet-script, and [ScrapingHub](https://github.com/scrapinghub) for Splash Server (as well as their respective teams and contributors).
