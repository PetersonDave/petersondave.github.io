---
layout: post
title:  "Scripting Media Library Reorganization with Sitecore Powershell Extensions"
date:   2017-07-21 20:00:00
categories: Sitecore PowerShell
comments: true
---
As part of integrating one Sitecore instance into another, reorganization was needed for proper placements of media library items. With two instances sharing some of the same media paths (think ```sitecore/media library/images```, for example), integrating one instance's media into another instance could become problematic. Best practice for media library item organization should assume site-specific paths and proper restrictions of image fields, anchored to specific media library folders. However, in this case, media library items were stored in more common item paths, opening the possibility of conflicts between the two instances.

The approach we took was to script the move of items to site-specific folders, ensuring no conflicts were encountered. The following scripts were executed in three steps:

1. Updating template source fields to point to a new path
2. Remove unnecessary path references in image fields (carry-over from a pre-Sitecore 7 deployment. This was deprecated in Sitecore 8)
3. Moving of the media library to the new path
	
# Step 1: Update Template Image Source Values

The following function accepts the name of a site as a parameter and uses that to append to a default ```/sitecore/Media Library/sites``` folder for site-specific media item storage. That value is then used to replace existing, more generic references:

{% highlight powershell %}
function Update-TemplateImageFieldSource {
    Param(
        [string]$siteName
        )

    Write-Host "Updating iamge field sources to point to updated paths"
    
    $templatesRoot = "master:\sitecore\templates\components"
    Set-Location -Path $templatesRoot
    Get-ChildItem -Recurse | foreach {
        $templateItem = $_
        $hasWrittenToHost = $false
        $itemPath = $_.FullPath
        $itemId = $_.Id
        
        $skipTemplate = $itemPath.Contains('templates/System') -or $itemPath.Contains('templates/Branches') -or $itemPath.Contains('templates/Modules')
        
        if (!$skipTemplate) {
            $isValidSourceValue = `
                $_.Type -eq 'Image' `
                -and $_._Source -ne $null `
                -and $_._Source -ne '' `
                -and !$_._Source.StartsWith('/sitecore/Media Library/sites/' + $siteName)
            
            if ($isValidSourceValue) {
                if (!$hasWrittenToHost) {
                    Write-Host $itemPath $itemId $_.Type -foregroundcolor "magenta"
                    $hasWrittenToHost = $true
                            
                    $newValue = $_._Source.Replace('/sitecore/Media Library','/sitecore/Media Library/sites/' + $siteName)
                            
                    if(!$isDryRun) {
                        $item.Editing.BeginEdit()
                        $_._Source = $newValue
                        $item.Editing.EndEdit()
                    }
                }
                        
                Write-Host 'Source' $_._Source
            }
        }
    }
}
{% endhighlight %}

# Step 2: Remove Unnecessary Path References

Running this script was not necessary for the movement of media, it was more of a clean-up task to eliminate path references to images that were about to be moved. While Sitecore has deprecated this method of item lookup, you can still see these values in the raw values of the image field. This method simple searches for all instances of mediapath and removed them.

{% highlight powershell %}
function Remove-MediaPathFromImageValues
{
    Write-Host "Removing media path from image field values..."
            
    $mediaMapPathRegEx = " mediapath=`".[^ ]*`""
    $contentRoot = "master:\sitecore\content"
    Set-Location -Path $contentRoot
    Get-ChildItem -Recurse | foreach {
        $item = $_
        $itemName = $_.FullPath
        $itemId = $_.Id
                
        $_.Fields | foreach {
            $fieldName = $_.Name
            $isImage = $_.Type.Equals('Image')
                    
            if ($isImage) {
                $hasMapPath = $_.Value -match $mediaMapPathRegEx
                    
                if ($hasMapPath) {
                    $oldValue = $_.Value
                    $newValue = $oldValue -replace $mediaMapPathRegEx, ''
        
                    Write-Host "updating $itemId $itemName [$fieldName]"
                    Write-Host "   from $oldValue"
                    Write-Host "    to $newValue"
                            
                    if (!$isDryRun) {
                        $item.Editing.BeginEdit()
                        $_.Value = $newValue
                        $item.Editing.EndEdit()
                                
                        Write-Host "  [update done]"
                    }
                }
            }
        }
    }
}
{% endhighlight %}

# Step 3: Moving Media Library Items to New Path

The final steps performs the movement of media library items to their new site-specific folder. 

{% highlight powershell %}
function Move-MediaLibrary {
    Param(
        [string]$siteNameToBeMoved,
        [string]$siteNameDestination
        )

    Write-Host "Moving media library images to site media library folder..."
            
    $mediasourceRootPath = "/sitecore/media library/sites/" + $siteNameToBeMoved
    $mediaSourceRootItem = Get-Item -Path $mediaSourceRootPath -ErrorAction SilentlyContinue
    if ($mediaSourceRootItem) {
        Set-Location -Path "master:\sitecore\media library"
        Get-ChildItem -Recurse | foreach {
            $isSystem = $_.FullPath -match "/sitecore/media library/system"
            $isAlreadyMoved = $_.FullPath -match "/sitecore/media library/sites/" + $siteNameToBeMoved
            $isDestinationSite = $_.FullPath -match "/sitecore/media library/sites/" + $siteNameDestination
            if ($isSystem -OR $isAlreadyMoved -OR $isDestinationSite) {
                Write-Host "Skipping system media library item" $_.FullPath -foregroundcolor "yellow"
            }
            else {
                $oldPath = $_.FullPath
                $newPath = $oldPath -replace "/sitecore/media library", $mediaSourceRootPath
                Write-Host $oldpath "to" $newpath -foregroundcolor "green"
                        
                if (!$isDryRun) {
                    Move-Item $_.FullPath -Destination $newpath
                }
            }
        }
    }
    else {
        Write-Host "site media library path does not exist"
    }
}
{% endhighlight %}