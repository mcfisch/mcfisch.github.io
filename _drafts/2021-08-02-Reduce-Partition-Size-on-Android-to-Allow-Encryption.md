---
title: "Reduce the data partition size on Android to allow encryption"
date: 2021-08-02 9:00 -0700
categories: [Android, Encryption]
tags: [android, encryption, filesystem]
excerpt: "On some Android devices (i.e. Moto G5 Plus a.k.a. `Potter`) device encryption doesn't work when installing a custom ROM, because the entire `/data` partition was used when it was formatted."
---

On some Android devices (i.e. Moto G5 Plus a.k.a. `Potter`) device encryption doesn't work when using custom ROMs, because the entire `/data` partition was used when it was formatted.

When (unsuccessfully) trying to encrypt the device's storage the output of `logcat` might show something like this:

```
08-11 16:39:39.687  2370  2379 D vold    : !e4crypt_is_native, spawning fdeEnableInternal
08-11 16:39:39.691  2370  5354 E Cryptfs : Bad magic for real block device /dev/block/platform/13540000.dwmmc0/by-name/USERDATA
08-11 16:39:39.691  2370  5354 E Cryptfs : Orig filesystem overlaps crypto footer region.  Cannot encrypt in place.
```

The Android FBE (File Based Encryption) needs some free space behind the partition to store some meta data to work, but there is none available. So in order to fix this the `/data` partion needs to be resized in a recovery session.

On newer devices that is possible with the `resize2fs` command (`resize2fs /dev/block/bootdevice/by-name/userdata <target block count>`), but not on older devices like `Potter`. Here the partion needs to be recreated manually with the proper size - accompanied by a *loss of data*. **This will wipe the device's storage!**

The process gneeds to take place in a recovery like [TWRP](https://twrp.me/), either in the integrated terminal, or through ADB - which for most people should be the more comfortable way.
On some older versions of TWRP the issue seems not to exist as the menu-controlled formatting will leave some space at the end of the partition, but at least on `v3.5.2_9` this isn't the case.

Note: **This will wipe your device, including the internal storage (`/data/media`) - so make sure to backup everything before following through with this procedure!**

## Find the correct device

First get the path of the actual block device or use the universal name `/dev/block/bootdevice/by-name/userdata`, either way will work:

```bash
~ # ls -l /dev/block/bootdevice/by-name/userdata | cut -d'>' -f 2
 /dev/block/mmcblk0p54
```

## Make sure the partion is unmounted

The device is usually mounted in two places. Both can be released in one step. If one isn't mounted the command will shoe a respective error that can be ignored.

```bash
~ # umount /data /sdcard
```

## Get partition and sector size

The current size of the partition as well as the block size can be read from the output of the file system check tool `fsck`, used with the device path as an argument:

```bash
~ # fsck.f2fs /dev/block/mmcblk0p54
Info: Segments per section = 1
Info: Sections per zone = 1
Info: sector size = 512
Info: total sectors = 112639967 (54999 MB)
Info: MKFS version
  "Linux version 3.18.79-lineage+ (jenkins@93f70f4836da) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Tue Apr 6 16:23:32 UTC 2021"
Info: FSCK version
  from "Linux version 3.18.79-lineage+ (jenkins@93f70f4836da) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Tue Apr 6 16:23:32 UTC 2021"
    to "Linux version 3.18.79-lineage+ (jenkins@93f70f4836da) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Tue Apr 6 16:23:32 UTC 2021"
Info: superblock features = 0 :
Info: superblock encrypt level = 0, salt = 00000000000000000000000000000000
Info: total FS sectors = 112639960 (54999 MB)
Info: CKPT version = 2
Info: checkpoint state = 5 :  compacted_summary unmount

[FSCK] Unreachable nat entries                        [Ok..] [0x0]
[FSCK] SIT valid block bitmap checking                [Ok..]
[FSCK] Hard link checking for regular file            [Ok..] [0x0]
[FSCK] valid_block_count matching with CP             [Ok..] [0x2]
[FSCK] valid_node_count matcing with CP (de lookup)   [Ok..] [0x1]
[FSCK] valid_node_count matcing with CP (nat lookup)  [Ok..] [0x1]
[FSCK] valid_inode_count matched with CP              [Ok..] [0x1]
[FSCK] free segment_count matched with CP             [Ok..] [0x6ab4]
[FSCK] next block offset is free                      [Ok..]
[FSCK] fixing SIT types
[FSCK] other corrupted bugs                           [Ok..]

Done.
```

Look for these lines to get the sector size and sector count:

```bash
~ # fsck.f2fs /dev/block/mmcblk0p54 | grep sector
Info: sector size = 512
Info: total sectors = 112639967 (54999 MB)
```

## Double-check the numbers

You can verify the numbers by multiplying them first:

```bash
~ # echo $((112639967*512))
57671663104
```

Then read the block count directly with the following command, which should give you the same result:

```bash
~ # blockdev --getsize64 /dev/block/mmcblk0p54
57671663104
```

## Calculate space to be freed up

Calculate the amount of disk space needed to reserve from the sector count, i.e. 32kB:

```bash
~ # echo $((32*1024/512))
64
```

For some devices 4kB might be enough, others require 16kB. I used 32kB to be on the safe side, and it doesn't make too much of a difference from the user perspective anyway. This particular device has about 53GB of storage available, so I surely won't notice the difference of a couple Kilobytes during regular use.

## Calculate the new size

Subtract above number from the Sector count

```bash
~ # echo $((112639967-64))
112639903
```

## Format partition with new the size

Now format the partition with the filesystem in this size:

```bash
~ # mkfs.f2fs /dev/block/mmcblk0p54 112639903

        F2FS-tools: mkfs.f2fs Ver: 1.7.0 (2016-07-28) [modified by Motorola to reserve space]

Info: Debug level = 0
Info: Label =
Info: Trim is enabled
Info: total device sectors = 112639967 (in 512 bytes)
Info: Segments per section = 1
Info: Sections per zone = 1
Info: sector size = 512
Info: total sectors = 112639903 (54999 MB)
Info: zone aligned segment0 blkaddr: 512
Info: format version with
  "Linux version 3.18.79-lineage+ (jenkins@93f70f4836da) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Tue Apr 6 16:23:32 UTC 2021"
Info: Discarding device: 112639903 sectors
Info: Secure Discarded 112639903 sectors
Info: Overprovision ratio = 0.860%
Info: Overprovision segments = 472 (GC reserved = 240)
Info: format successful
```

If the commands from above are run again they should now reflect the new partition size.

## Setup and enxrypt the device

Now you can set up the device again, then gp ahead and try encrypting it again. There should no longer be an error, the device encryption should finish just fine after an autmatic reboot or two.
