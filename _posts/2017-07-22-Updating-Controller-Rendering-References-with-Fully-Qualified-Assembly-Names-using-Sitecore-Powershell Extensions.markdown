---
layout: post
title:  "Updating Controller Rendering References with Fully Qualified Assembly Names using Sitecore Powershell Extensions"
date:   2017-07-22 20:00:00
categories: Sitecore PowerShell
comments: true
---

With a multisite Sitecore instance in Sitecore 8,1u3, we were in need of having site-specific renderings, some of which shared controller names across sites. While integrating on instance into another, scripting was required to update Sitecore controller rendering references to use their fully qualified assembly names instead of the more simple approach. The following script was used to perform this update.

The script assumes controller renderings reside in their site's area, as well as, having a default namespace preceding the area reference in the namespace.

{% highlight powershell %}
function Update-ControllerRenderingsWithNamespace {
     Param(
        [string]$defaultNamespace,
        [string]$siteName
        )
    $startLocation = 'master:\sitecore\layout\Renderings'
    Set-Location $startLocation
        
    Get-ChildItem -recurse | where-object { $_.TemplateName -match 'Controller rendering' -and !($_."Controller" -match ', $defaultNamespace') } | foreach {
        $controller = $_."Controller".Trim()
        $controllerValue = "$($defaultNamespace).Areas.$($siteName).Controllers.$($controller)Controller, $($defaultNamespace)"
        
        Write-Host "Found" $_.FullPath
        Write-Host "Updaing to: " $controllerValue
        $_.BeginEdit()
        $_."Controller" = $controllerValue
        $_.EndEdit()
    }
}
{% endhighlight %}