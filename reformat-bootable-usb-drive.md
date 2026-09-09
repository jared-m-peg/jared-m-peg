## REFORMATTING A (BOOTABLE) USB THUMB DRIVE IN COMMAND PROMPT ON WINDOWS 10
This guide is for anyone who is trying to reformat a bootable USB thumb drive in Windows Explorer, but it does not work and reports that it cannot be formatted and is write-protected. This guide will show you how to reformat the USB drive.

Run cmd.exe as Administrator

Run the following command:
```
diskpart
```

List all disks (internal SSD, attached USB drives):
```
list disk
```

It will output something like this:
Disk 0    Online          238 GB  1024 KB        *
Disk 1    Online           14 GB      0 B        *

Disk 1 is my Toshia USB thumb drive (16 GB), Disk 0 is my internal SSD (240 GB).

```
select disk 1
```

It will report:
Disk 1 is now the selected disk

```
attributes disk
```

It will report:
Current Read-only State : No
Read-only  : No
Boot Disk  : No
Pagefile Disk  : No
Hibernation File Disk  : No
Crashdump Disk  : No
Clustered Disk  : No

It should display:
Read-only : No

```
attributes disk clear readonly
```

It should report:
Disk attributes cleared successfully.

```
disk clean
```

It should report:
DiskPart succeeded in cleaning the disk.

Convert to MBR partitioning. The process used to make the USB drive bootable may have converted it to GPT partitioning. To convert it to MBR partitioning use the following command: 
```
convert mbr
```

```
create partition primary
```
It should report:
DiskPart succeeded in creating the specified partition.

```
select partition 1
```
It should report:
Partition 1 is now the selected partition.

```
format fs=fat32 quick
```
It should report:
100 percent completed
DiskPart successfully formatted the volume.

```
assign
```
It should report:
DiskPart successfully assigned the drive letter or mount point.

```
exit
```

The USB thumb drive is now formatted. Safely remove it.
You can run diskmgmt.msc as Administrator to check if your USB thumb drive is "Healthy, Active, Primary Partition". It may have displayed "Basic Data Partition" before these steps were taken to reformat it.


