# Setting Integrity Streams After or While Formatting a Volume

There are 2 ways to set Integrity Streams on Windows volumes:

## when formatting the volume using ```Format-Volume```

```powershell
# First we get the disk number that we wish to format in a variable (same Disk <number> you see on the Disk Management Console)
$DiskToFormat = Get-Disk 3

#Then we get the partition to format in a variable
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2
#Note we're formatting partition #2, as #1 is usually reserved when a new partition is created in GPT type.

#Format-Volume is the only way to set Integrity Streams on the whole volume. After this you will need to provide a
# drive letter or volume mountpoint root and use Set-FileIntegrity
Format-Volume -Partition $PartitionToFormat –AllocationUnitSize 65536 –FileSystem REFS –NewFileSystemLabel “ExVolXX" –SetIntegrityStreams:$false -confirm:$false
```

## or after formatting, using ```Set-FileIntegrity```

You can use ```Get-FileIntegrity``` on the volume to check if the "IntegrityStreams" is enabled or not. For the whole volume, you'll have to run ```Get/Set-FileIntegrity``` on the root volume.
For this, it's easier to get if the volume has a letter or mountpoints assigned to it, **BUT** you can also check the volume root using the Volume ID.

To get the volume ID, you need to get the partition you want to check in Powershell, which is achieved using the same commands we used above when we formatted the volume:

```powershell
# Get the disk number in a variable
$DiskToCheck = Get-Disk 3
#Then get the partition corresponding to that disk, take partition nb #2 because ReFS uses a reserved space for the first partition
$PartitionToCheck = Get-Partition -DiskNumber $DiskToCheck.Number -PartitionNumber 2
```

And to get the volume root access point to use with Get/Set-FileIntegrity:

```powershell
$PartitionToCheck.AccessPaths
```

If you didn't assign mountpoints or letters to that volume, you'll only see the volume ID like the below:

```powershell
[PS] C:\>$PartitionToFormat.AccessPaths
\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\
```

And finally, to check if Integrity Streams is enabled or not, you just need to run Get/Set-FileIntegrity on the above access path (or any existing access point if you have some):

```powershell
Get-FileIntegrity "\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\"
```

And if it's disabled, the output will look like the below on the "Enabled" property:
```powershell
[PS] C:\>Get-FileIntegrity "\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\"

FileName                                          Enabled Enforced
--------                                          ------- --------
\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\ False   True    
```

You can enable or disable Integrity Streams using Set-FileIntegrity on that volume:

```powershell
Set-FileIntegrity "\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\" -Enable $False
```

If you don't want to copy/paste or type the whole volume number, you can use the PowerShell variable you used before where you stored the Partition object (```$PartitionToCheck```) and call the ```accesspaths``` property of that object:

```powershell
#Just pasting again the command sequence to store the partition object in a PowerShell variable:
$DiskToCheck = Get-Disk 3
$PartitionToCheck = Get-Partition -DiskNumber $DiskToCheck.Number -PartitionNumber 2

#Then to check the Integrity Streams status:
Get-FileIntegrity $PartitionToCheck.AccessPaths[0]

#And to set the integrity streams status:
Set-FileIntegrity $PartitionToCheck.AccessPaths[0] -Enable $false
```


# Bonus - Adding a volume Mount Point a volume using PowerShell

## Appearance on the Disk Management Console

After formatting the volume with PowerShell, you can assign a drive letter or a volume mount point to the volume. You can do so using PowerShell as well, or using the Disk Management Console:

<img src=https://user-images.githubusercontent.com/33433229/123560522-b224d780-d770-11eb-80f0-d7f1a4e565d2.png Width = 400>

and then:

<img src=https://user-images.githubusercontent.com/33433229/123560647-9a9a1e80-d771-11eb-8a43-b23ca46afead.png width = 200>


## And the equivalent in PowerShell

And in PowerShell that would be like:

```powershell
#Just taking back the variable defined above before formatting, as it will design the same partition we've been working with 
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2

# Specifying that we don't want to add a simple drive letter for that partition (piping the #PartitionToFormat variable into Set-Partition)
$PartitionToFormat | Set-Partition -NoDefaultDriveLetter:$True

#Adding the mountpoint using the #partitionToFormat variable into Add-PartitionAccessPath
# NOTE: the NFTS folder must exist before we can assign it
$PartitionToFormat | Add-PartitionAccessPath $PartitionToFormat -AccessPath "C:\ExchangeVolumes\ExVolXX"-Passthru
```

## How to check that it worked using PowerShell ?

Simply check the property named "AccessPaths" of the Partition object, like we did earlier:
NOTE: don't forget to refresh the $PartitionToFormat variable otherwise you'll see the Access Paths of the partition before you actually assigned an access path to it:

```powershell
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2
$PartitionToFormat.AccessPaths
```
And you'll get an output like the below:

```powershell
[PS] C:\>$PartitionToFormat.AccessPaths
C:\ExchangeVolumes\ExVolXX\
\\?\Volume{5fc9932d-51d2-4dcb-ba25-7eb5fb6648ba}\
```

# Annex - tests on files created before or after setting File Integrity on volumes

```powershell
#Disabling File Integrity at the root level
Set-FileIntegrity C:\ExchangeVolumes\ExVolXX\ -Enable $false
#File Integrity now disabled

#Testing test.txt created *BEFORE* disabling file integrity on the root:


Get-FileIntegrity C:\ExchangeVolumes\ExVolXX

<####### OUTPUT

[PS] C:\>Get-FileIntegrity C:\ExchangeVolumes\ExVolXX

FileName                   Enabled Enforced
--------                   ------- --------
C:\ExchangeVolumes\ExVolXX False   True  

#>


Get-FileIntegrity C:\ExchangeVolumes\ExVolXX\test.txt
<#RESULT

[PS] C:\>Get-FileIntegrity C:\ExchangeVolumes\ExVolXX\test.txt

FileName                            Enabled Enforced
--------                            ------- --------
C:\ExchangeVolumes\ExVolXX\test.txt True    True

==> STILL ENABLED :-(

#>

#Creating new file *AFTER* disabling file integrity on the root:
Get-FileIntegrity C:\ExchangeVolumes\ExVolXX\test-createdAFTERdisablingFileIntegCheck.txt

<#RESULT
[PS] C:\>Get-FileIntegrity C:\ExchangeVolumes\ExVolXX\test-createdAFTERdisablingFileIntegCheck.txt

FileName                                                                Enabled Enforced
--------                                                                ------- --------
C:\ExchangeVolumes\ExVolXX\test-createdAFTERdisablingFileIntegCheck.txt False   True  

==> Integrity DISABLED !! :-)

#>

```

**CONCLUSION: All files created AFTER File Integrity disabled on the root have their integrity DISABLED
All files created BEFORE file integrity disabled will have their integritythat remain ENABLED**
