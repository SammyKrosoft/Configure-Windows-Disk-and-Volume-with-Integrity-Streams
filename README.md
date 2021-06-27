# Setting Integrity Streams

There are 2 ways to set Integrity Streams on Windows volumes:

- when formatting the volume using Format-Volume

```powershell
# First we get the disk number that we wish to format in a variable (same Disk <number> you see on the Disk Management Console)
$DiskToFormat = Get-Disk 3

#Then we get the partition to format in a variable
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2
#Note we're formatting partition #2, as #1 is usually reserved for chechsums when using ReFS

#Format-Volume is the only way to set Integrity Streams on the whole volume. After this you will need to provide a
# drive letter or volume mountpoint root and use Set-FileIntegrity
Format-Volume -Partition $PartitionToFormat –AllocationUnitSize 65536 –FileSystem REFS –NewFileSystemLabel “ExVolXX" –SetIntegrityStreams:$false -confirm:$false
```

After formatting the volume with PowerShell, you can assign a drive letter or a volume mount point to the volume. You can do so using PowerShell as well, or using the Disk Management Console:

<img src=https://user-images.githubusercontent.com/33433229/123560522-b224d780-d770-11eb-80f0-d7f1a4e565d2.png Width = 400>

and then:

<img src=https://user-images.githubusercontent.com/33433229/123560647-9a9a1e80-d771-11eb-8a43-b23ca46afead.png width = 200)


And in PowerShell that would be like:

```powershell
#Just taking back the variable defined above before formatting, as it will design the same partition we''ve been working with 
$PartitionToFormat = Get-Partition -DiskNumber $DiskToFormat.Number -PartitionNumber 2

# Specifying that we don''t want to add a simple drive letter for that partition (piping the #PartitionToFormat variable into Set-Partition)
$PartitionToFromat | Set-Partition -NoDefaultDriveLetter:$True

#Adding the mountpoint using the #partitionToFormat variable into Add-PartitionAccessPath
# NOTE: the NFTS folder must exist before we can assign it
$PartitionToFormat | Add-PartitionAccessPath $PartitionToFormat -AccessPath "C:\ExchangeVolumes\ExVolXX"-Passthru
```

