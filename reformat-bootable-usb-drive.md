## REFORMATTING A (BOOTABLE) USB THUMB DRIVE IN COMMAND PROMPT ON WINDOWS 10
This guide is for anyone who is trying to reformat a bootable USB thumb drive in Windows Explorer, but it does not work, and reports that it cannot be formatted and is write-protected. This guide shows you how to reformat the USB drive.

First, run a Command Prompt (cmd.exe) as an Administrator.

In the Command Prompt window, run the following command:
```cmd
diskpart
```

List all disks (internal SSD, attached USB drives):
```cmd
list disk
```

It will output something like this:
```text
Disk 0    Online          238 GB  1024 KB        *
Disk 1    Online           14 GB      0 B        *
```
Disk 1 is my Toshiba USB thumb drive (16 GB), Disk 0 is my internal SSD (240 GB).

Now run:
```cmd
select disk 1
```

It will report:
```text
Disk 1 is now the selected disk
```

Run:
```cmd
attributes disk
```
It will report:
```text
Current Read-only State : No
Read-only  : No
Boot Disk  : No
Pagefile Disk  : No
Hibernation File Disk  : No
Crashdump Disk  : No
Clustered Disk  : No
```
It should display:
```text
Read-only : No
```

Run:
```cmd
attributes disk clear readonly
```
It should report:
```text
Disk attributes cleared successfully.
```

Run:
```cmd
disk clean
```

It should report:
```text
DiskPart succeeded in cleaning the disk.
```

Convert to MBR partitioning. The process used to make the USB drive bootable may have converted it to GPT partitioning. To convert it to MBR partitioning use the following command: 
```cmd
convert mbr
```

```cmd
create partition primary
```
It should report:
```text
DiskPart succeeded in creating the specified partition.
```

Run:
```cmd
select partition 1
```
It should report:
```text
Partition 1 is now the selected partition.
```
Run:
```cmd
format fs=fat32 quick
```
It should report:
```text
100 percent completed
DiskPart successfully formatted the volume.
```

Run:
```cmd
assign
```
It should report:
```text
DiskPart successfully assigned the drive letter or mount point.
```

Run:
```cmd
exit
```

The USB thumb drive is now formatted. Safely remove it.
You can run diskmgmt.msc as Administrator to check if your USB thumb drive is "Healthy, Active, Primary Partition". It may have displayed "Basic Data Partition" before these steps were taken to reformat it.


