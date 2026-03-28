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




# Failed disk in ZFS Pool

## Procedure to replace the disk

Note down the serial number of the device which is failing.
You should have labeled them on the trays for easier identification. Because if not, it is a pain to recover which disk is on what SATA port and identify the tray !

Power off the machine and remove the disk. Replace the disk.
If you are confident with the hardware and have a hotplug disk tray, then you can straight remove the failing drive and replace it.

After the server is restarted, the useful command is `zpool status`

```
zpool status


root@okarin:~# zpool status
  pool: rpool
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
	The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
	the pool may no longer be accessible by software that does not support
	the features. See zpool-features(7) for details.
  scan: scrub repaired 0B in 00:00:04 with 0 errors on Sun Mar  8 00:24:05 2026
config:

	NAME                                                                                         STATE     READ WRITE CKSUM
	rpool                                                                                        ONLINE       0     0     0
	  mirror-0                                                                                   ONLINE       0     0     0
	    nvme-nvme.1e4b-30303334333835303030303235-5847373030302d3531322032323830-00000001-part3  ONLINE       0     0     0
	    nvme-nvme.1e4b-30303334303837303030313538-5847373030302d3531322032323830-00000001-part3  ONLINE       0     0     0

errors: No known data errors

  pool: vm-pool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
  scan: resilvered 2.53G in 00:00:26 with 0 errors on Mon Mar  9 19:09:36 2026
config:

	NAME                                    STATE     READ WRITE CKSUM
	vm-pool                                 DEGRADED     0     0     0
	  raidz1-0                              DEGRADED     0     0     0
	    ata-SSH_512GB_AA000000000000000648  ONLINE       0     0     0
	    ata-SSD_512GB_AA000000000000001172  ONLINE       0     0     0
	    ata-SSD_512GB_SN-on-the-lable       ONLINE       0     0     0
	    9017512507357970393                 UNAVAIL      0     0     0  was /dev/disk/by-id/ata-SSD_512GB_AA10040300125000742-part1
```

Identify the missing disk in the pool you are working on. `UNAVAIL`
Also take care of the `DISK_ID` at the begining of the line. It will be usefull later.
Identify your new disk with `ls -la /dev/disk/by-id/`
something like :
```
/dev/disk/by-id/ata-P3-512_9F51126332721
```

The you'll have to use the `zpool replace` command.
```
zpool replace vm-pool 9017512507357970393 /dev/disk/by-id/ata-P3-512_9F51126332721
```
- First arg is the zfs pool name  
- Second arg is the `DISK_ID` of the impacted disk in the pool
- Third is the new disk path

After that the new disk will be use in the pool the replace the previous one and ZFS will go to resilvering process

```
root@okarin:~# zpool status
  pool: rpool
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
	The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
	the pool may no longer be accessible by software that does not support
	the features. See zpool-features(7) for details.
  scan: scrub repaired 0B in 00:00:04 with 0 errors on Sun Mar  8 00:24:05 2026
config:

	NAME                                                                                         STATE     READ WRITE CKSUM
	rpool                                                                                        ONLINE       0     0     0
	  mirror-0                                                                                   ONLINE       0     0     0
	    nvme-nvme.1e4b-30303334333835303030303235-5847373030302d3531322032323830-00000001-part3  ONLINE       0     0     0
	    nvme-nvme.1e4b-30303334303837303030313538-5847373030302d3531322032323830-00000001-part3  ONLINE       0     0     0

errors: No known data errors

  pool: vm-pool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Mar 28 17:41:59 2026
	10.3G / 10.3G scanned, 74.4M / 8.29G issued at 10.6M/s
	15.6M resilvered, 0.88% done, 00:13:11 to go
config:

	NAME                                    STATE     READ WRITE CKSUM
	vm-pool                                 DEGRADED     0     0     0
	  raidz1-0                              DEGRADED     0     0     0
	    ata-SSH_512GB_AA000000000000000648  ONLINE       0     0     0
	    ata-SSD_512GB_AA000000000000001172  ONLINE       0     0     0
	    ata-SSD_512GB_SN-on-the-lable       ONLINE       0     0     0
	    replacing-3                         DEGRADED     0     0     0
	      9017512507357970393               UNAVAIL      0     0     0  was /dev/disk/by-id/ata-SSD_512GB_AA10040300125000742-part1
	      ata-P3-512_9F51126332721          ONLINE       0     0     0  (resilvering)

errors: No known data errors
```

After a while depending of the volume of datas, the volume will become ONLINE and everything will be clear.

If using a NAS with `smartctl` don't forget to update smartd.conf and restart service.

