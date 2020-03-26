# Failed disk in md Array

## Procedure to replace the disk
- First, write all cache to disk, not mandaroty

`sync`

- Mark the disk as failed, maybe the disk is already marked as failed

`mdadm --manage /dev/mdZ --fail /dev/sdXY`

- Verify by :
```
cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid5] [raid4] [raid6] [raid10]
md0 : active raid1 sda1[0] sdb1[2](F)
      976773168 blocks [2/1] [U_]

md1 : active raid1 sda2[0] sdb2[1]
      976773168 blocks [2/2] [UU]
```

- Remove the disk from array

`mdadm --manage /dev/mdZ --remove /dev/sdXY`

- Replace the disk physically

- Clone the disk partition from a member of the array
```
sgdisk /dev/sdX -R /dev/sdY 
! Be careful /dev/sdX is the source
! /dev/sdY is the new disk
! DO NOT MIX UP
```

- Re-add the new disk to array

`mdadm --manage /dev/mdZ --add /dev/sdXY`

- Verify by :
```
/sbin/mdadm --detail /dev/mdZ
cat /proc/mdstat
```


## Inspect disk afterward
After the disk was marked as failed and safely replaced, data are safe.
You may take some time to inspect the drive and decide whether is goes to trash or not.

! Be Carefull

! After this procedure, all data are lost

`smartctl -t long /dev/sdX`

If okay at this point continue with wiping disk.
Get Physical sector size
```
fdisk -l /dev/sdX
Disk /dev/sdX: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: ST3500413AS
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
...
Device     Boot      Start        End         Sectors     Size  Id Type
/dev/sdX1            2048         3839711231  3839709184  1,8T  83 Linux
/dev/sdX2            3839711232   3907029167  67317936    32,1G  5 Extended
```

Next, wipe the disk
```
dd if=/dev/zero of=/dev/sdX bs=4096[physical sector size] status=progress
or
dd if=/dev/urandom of=/dev/sda2 bs=4096[physical sector size] status=progress
```
