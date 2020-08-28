# Failed disk in md Array

## Procedure to replace the disk
- First, write all cache to disk, not mandatory

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
! BE CAREFUL
! /dev/sdX is the source
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

```
! BE CAREFUL
! After this procedure, all data are lost
```

`smartctl -g all`

`smartctl -s apm=254`

`smartctl -C -t long /dev/sdX` or `smartctl -C -t select,0-max /dev/sdX`

`smartctl -l selftest /dev/sdX`


If okay at this point continue with wiping disk.

```
dd if=/dev/zero of=/dev/sdX bs=1M status=progress
```



futur:

If you know the array UUID, then mdadm --assemble /dev/md0 --uuid <uuid> (note the slight difference in parameter order) will do what you want: scan all unused volumes for ones that have md metadata for the given UUID. Other options:

mdadm --assemble /dev/md0 --name <name> (does the same thing as --uuid, but with an array name instead of a UUID.)
mdadm --assemble /dev/md0 --super-minor <minor id #> (does the same thing as --uuid, but with minor device numbers in the metadata. Only recommended for version 0.90 metadata.)
mdadm --assemble /dev/md0 /dev/disk/by-id/<disk>... (if udev has set up /dev/disk/by-id aliases, which should be static across hardware changes.)
mdadm --assemble --scan with no arrays listed in the configuration file (scan all unused volumes for md metadata, and assemble RAID arrays based on what's found. Note that if you've got multiple arrays and only want to set up one of them, or if your array has gotten split, this won't do what you want.)
