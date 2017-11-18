---
layout: post
title:  "Generating New Item Ids For Existing Items with Sitecore Powershell Extensions"
date:   2017-07-23 20:00:00
categories: Sitecore PowerShell
comments: true
---

We had a very unique situation in that we were integrating one Sitecore instance into another, where items between the two instances shared the same guid. In other words, both sites shared the same exact Sitecore starting point. Media library folders, content asset folders, etc., all have the same item guids between Sitecore instances. Integrating one instance into another wasn't as simple as it first appeared, we couldn’t simple drop one into the other and expect it to work without any issues. In test runs, we experienced errors due to conflicts in these areas on item creation. We were using Unicorn to serialize to disk form one instance and deserialize back into the other.

To get past these commonalities between item ids, the following script was used to eliminate these conflicts by generating new items and moving content over to the newly generated items:

1. Create the new item with a temp name
2. Update field values
3. Move all child items to the new item
4. Rename the original item
5. Rename the new item to the original name

Note that this script does not accommodate for moving of referrers of the original item to the new one. A separate process was used to resolve those issue before removing the original items.

{% highlight powershell %}
function Update-MoveItemToNewItemGuid {
    Param(
        [string]$pathToMove,
        [bool]$isBucket
        )
    
    $itemToMove = Get-Item -Path $pathToMove
    if (!$itemToMove) {
        throw "$itemToMove doesn't exist"
    }
    
    $pathNew = "$($pathToMove)-new"
    
    Write-Host "testing if item path $pathNew already exists"
    
    $itemNewPath = Get-Item -Path $pathNew
    if ($itemNewPath) {
        throw "Item already exists at $pathNew"
    }
    
    Write-Host "Unprotecting original item..."
    Unprotect-Item -Path $pathToMove
    
    $itemNewTemplatePath = $itemToMove.Template.InnerItem.Paths.FullPath
    Write-Host "Step 1. Item does not exist. Creating the item $pathNew"
    Write-Host "New-Item -Path $pathNew -ItemType $itemNewTemplatePath" -foregroundcolor "magenta"
    $newItem = New-Item -Path $pathNew -ItemType $itemNewTemplatePath
    
    Write-Host "Step 2. Cycle through item fields, copying values"
    $newItem.Fields.ReadAll()
    $itemToMove.Fields.ReadAll()
    $newItem.Fields | foreach {
        if ($itemToMove.Fields[$_.Name].Value -ne $newItem.Fields[$_.Name].Value) {
            Write-Host "updating field value to: " $itemToMove.Fields[$_.Name].Value "from: " $newItem.Fields[$_.Name].Value
                
            $newItem.Editing.BeginEdit()
            $newItem.Fields[$_.Name].Value = $itemToMove.Fields[$_.Name].Value
            $newItem.Editing.EndEdit()
        }
    }
    
    Write-Host "Step 3. Moving all child items"
        
    if ($isBucket) {
        Get-ChildItem $pathToMove -recurse | foreach {
            # skip bucket folders
            if ($_.TemplateName -eq 'Bucket') {
                # create bucket folders with new guids
                Write-Host $_.Paths.FullPath
                Write-Host $itemToMove.Paths.FullPath
                Write-Host $newItem.Paths.FullPath
                    
                $childNewPath = $_.Paths.FullPath -replace $itemToMove.Paths.FullPath, $newItem.Paths.FullPath
                    
                Write-Host "Creating new bucket folder in $childNewPath"
                New-Item -Path $childNewPath -ItemType $_.Template.InnerItem.Paths.FullPath
            }
            else {
                # move the actual item
                $itemName = $_.Name
                $childOldPath = $_.Paths.FullPath
                    
                # set new path to parent root. Let Sitecore handle creation of bucket folders
                $childNewPath = $_.Paths.FullPath -replace $itemToMove.Paths.FullPath, $newItem.Paths.FullPath
        
                Write-Host "Moving $childOldPath to $childNewPath"
                Write-Host "Move-Item -Path $childOldPath -Destination $childNewPath" -foregroundcolor "magenta"
                Move-Item -Path $childOldPath -Destination $childNewPath
            }
        }
    }
    else {
        Get-ChildItem $pathToMove | foreach {
            $itemName = $_.Name
            $childOldPath = $_.Paths.FullPath
            $childNewPath = "$pathNew\$itemName"
    
            Write-Host "Moving $childOldPath to $childNewPath"
            Write-Host "Move-Item -Path $childOldPath -Destination $childNewPath" -foregroundcolor "magenta"
            Move-Item -Path $childOldPath -Destination $childNewPath
        }
    }
            
    Write-Host
        
    Write-Host "Step 4. Verifying the original item doesn't have any children"
    if ($itemToMove.HasChildren) {
        throw "Original item still has child items! Aborting update."
    }
        
    $backupPath = "$($pathToMove)-old"
    Write-Host "Step 5. Moving item from $pathToMove to $backupPath"
    Write-Host "Move-Item -Path $pathToMove -Destination $backupPath" -foregroundcolor "magenta"
    Move-Item -Path $pathToMove -Destination $backupPath
    
    Write-Host "Step 6. Moving the item back to original position from $pathNew to $pathToMove"
    Write-Host "Move-Item -Path $pathNew -Destination $pathToMove" -foregroundcolor "magenta"
    Move-Item -Path $pathNew -Destination $pathToMove
}
{% endhighlight %}