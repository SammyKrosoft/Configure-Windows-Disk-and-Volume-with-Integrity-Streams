# Setting Integrity Streams After or While Formatting a Volume

## Prior to format the disk if the new partition is not created yet you can use powershell or diskmgmt.msc to create a new one

> **NOTE**: If a partition is already created, move to next section. 

You can create a new partition (usually GPT for Exchange databases volumes) using the Disk Management console (not fully covered here but very easy as in right-click "New simple volume") or using PowerShell.

The management console's new volume creation would look like:

<img src=https://user-images.githubusercontent.com/33433229/123822287-245cff80-d8ca-11eb-8069-c14b009b14d7.png width = 300>

then the screens sequence :

<img src=https://user-images.githubusercontent.com/33433229/123822403-3f2f7400-d8ca-11eb-84ac-f39a7a46154a.png width = 200>

<img src=https://user-images.githubusercontent.com/33433229/123822496-59695200-d8ca-11eb-9975-0751355d56d6.png width = 200>

<img src=https://user-images.githubusercontent.com/33433229/123822545-62f2ba00-d8ca-11eb-9f24-5e96f283abb9.png width = 200>

> NOTE: don't assign a drive letter or drive path yet, we'll do it later (I like baby steps to ensure no mistakes are made)

You can format the disk now (choosing ReFS, and **not** NTFS), but if you do so you'll have to use ```Set-FileIntegrity``` later to ensure Integrity Streams is disabled. I prefer not formatting it now and use ```Format-Volume``` to be able to set the Integrity Streams during the disk formatting.
<img src=https://user-images.githubusercontent.com/33433229/123823004-c7157e00-d8ca-11eb-83e1-006a919dbd74.png width = 200>

With PowerShell, the sequence would be like the below:

1- We get the disk to check and put it in a variable	

```powershell
$DiskToSetup = Get-Disk 3  
```

2- We check the disk information

```powershell
$DiskToSetup | ft -a 
```

3a- If the disk has not been initialized yet, we run this:	
```powershell
Initialize-Disk $DiskToSetup.Number -PartitionStyle GPT
```

3b- If the disk has been initialized but not in MBR, we can run this
```powershell
Set-Disk -Number $DiskToSetup.Number -PartitionStyle GPT 
```

4- finally, you can create a new partition
```powershell
New-Partition -UseMaximumSize -DiskNumber $DiskToSetup.Number
```

Then, there are 2 ways to set Integrity Streams on Windows volumes:
- setting it while formatting the volume
- setting it after the volume is formatted

## Setting Integrity Streams While formatting the volume using ```Format-Volume```

> **Note**: Disk partitionned using GPT partition (GPT stands for Globally Unique Identifier or GUID partition table) always have an additional partition that's a reserved space of 32MB or 128MB depending on the disk size 16GB disks and less : reserved partition is 32MB, otherwise it's 128MB. This space is referred to as the Microsoft Reserved partition or MSR, and is usually identified by Windows PowerShell as Partition #1, while the usable partition you create is referred as Partition #2.
>
>- [More information here about GPT partitions](https://www.diskpart.com/articles/gpt-reserved-partition-128mb.html)
>
>- [Microsoft doc about GPT partitions ](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-and-gpt-faq)

```powershell
# First we get the disk number that we wish to format in a variable (same Disk <number> you see on the Disk Management Console)
$DiskToFormat = Get-Disk 3

#Then we get the partition to format in a variable
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2
#Note we're formatting partition #2, as #1 is usually reserved when a new partition is created in GPT type.

#The -SetIntegrityStreams parameter is used to enable or disable file integrity streams feature for the whole volume with the Format-Volume cmdlet.
Format-Volume -Partition $PartitionToFormat –AllocationUnitSize 65536 –FileSystem REFS –NewFileSystemLabel “ExVolXX" –SetIntegrityStreams:$false -confirm:$false
```

## Setting Integrity Streams after formatting, using ```Set-FileIntegrity```

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

And finally, to check if Integrity Streams is enabled or not, you just need to run ```Get/Set-FileIntegrity``` on the above access path (or any existing access point if you have some):

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

If you don't want to copy/paste or type the whole volume GUID, you can use the PowerShell variable you used before where you stored the Partition object (```$PartitionToCheck```) and call the ```AccessPaths``` property of that object - note that this property is an array, and in PowerShell the first index of an array is 0, hence the notation ```$partitionToCheck.AccessPaths[0]``` that points to the raw path to the partition that I used in the above 
```Get/Set-FileIntegrity``` examples if the volume doesn't have any other mountpoints and drive letters:

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
#Just pasting again the command sequence to store the partition object in a PowerShell variable:
$DiskToCheck = Get-Disk 3
$PartitionToCheck = Get-Partition -DiskNumber $DiskToCheck.Number -PartitionNumber 2

# Specifying that we don't want to add a simple drive letter for that partition (piping the #PartitionToFormat variable into Set-Partition)
$PartitionToCheck | Set-Partition -NoDefaultDriveLetter:$True

#Adding the mountpoint using the $PartitionToFormat variable into Add-PartitionAccessPath
# NOTE: the NFTS folder must exist before we can assign it
$PartitionToCheck | Add-PartitionAccessPath -AccessPath "C:\ExchangeVolumes\ExVolXX"-Passthru
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
