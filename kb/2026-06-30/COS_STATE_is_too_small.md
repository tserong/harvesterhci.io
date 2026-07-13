---
title: Upgrade issues with 8G COS_STATE partition
description: 'Understading and resolving OS image corruption issues that can occur during upgrade if the COS_STATE partition is only 8G in size, rather than the expected 15G'
slug: upgrade_isses_with_8g_cos_state_partition
authors:
  - name: Tim Serong
    title: Principal Software Engineer
    url: https://github.com/tserong
    image_url: https://github.com/tserong.png
tags: [operating system, upgrade]
hide_table_of_contents: false
---

When upgrading from SUSE Virtualization v1.7 to v1.8, you will encounter an operating system image corruption issue if _all_ of the following conditions are met:

- Harvester v1.4.1 or earlier was originally installed.
- A separate data disk is used.
- The cluster has been continually upgraded in place.

The root cause is [a bug in Harvester v1.4.1 and earlier](https://github.com/harvester/harvester/issues/7493). If the system was originally configured to use a separate data disk, this bug caused the `COS_STATE` partition size to be set to 8G at installation time, rather than the expected 15G. This lack of space causes trouble during subsequent upgrades. For example, if you upgrade from v1.7.x to v1.8.x, the `active.img` file that stores the operating system image, [will corrupt and result in unexpected system failures](https://github.com/harvester/harvester/issues/10687). 

If one of the following conditions are met, you can safely ignore this report:

- If you initially installed Harvester v1.4.2 or newer, the `COS_STATE` partition will be 15G and this problem will not occur.
- If you initially installed Harvester v1.4.1 or earlier, but deployed with a shared data/installation disk (rather than a separate data disk), the `COS_STATE` partition will be 15G and this problem will not occur.
- If you have re-installed or added any new nodes with Harvester v1.4.2 or newer, the `COS_STATE` partition will be 15G and this problem will not occur on those nodes.

## Checking if your system is affected

To determine if your system is affected, check the size of the `COS_STATE` partition on all nodes.

This can be done manually by logging in via `ssh`, then using `lsblk` to check the partition sizes. For example:

```
# lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT
sda      250G
├─sda1    64M vfat   COS_GRUB
├─sda2    64M ext4   COS_OEM         /oem
├─sda3     4G ext4   COS_RECOVERY
├─sda4     8G ext4   COS_STATE       /run/initramfs/cos-state
└─sda5 237.9G ext4   COS_PERSISTENT  /usr/local
sdb      500G ext4   HARV_LH_DEFAULT /var/lib/harvester/defaultdisk
```

Output example of an _affected_ node:
```
# lsblk -o NAME,SIZE,LABEL | grep COS_STATE
├─sda4     8G COS_STATE
```

Output example of an _unaffected_ node:
```
# lsblk -o NAME,SIZE,LABEL | grep COS_STATE
├─sda4     15G COS_STATE
```

If you run the latest version of the [pre-check script](https://github.com/harvester/upgrade-helpers/tree/main/pre-check) on a v1.7.x control plane node, this will automatically check all nodes. When run on an affected system the output will be similar to the following:

(**TBD: https://github.com/harvester/upgrade-helpers/pull/50 needs merging before this will work**)

```
# ./check.sh
Upgrading from version v1.7.1

[...]

Starting COS_STATE partition size check....
Waiting for COS_STATE-Size validator to finish on all nodes...
COS_STATE-Size FAILED:
[pod/upgrade-helper-cos-state-size-check-cvzj4/validator] ERROR: harvester-node-1: COS_STATE partition is only 8GB, upgrade to Harvester v1.8.x will result in corrupt OS image. Suggest re-installing first to ensure partition is 15GB.
[pod/upgrade-helper-cos-state-size-check-cvzj4/validator] RESULT: harvester-node-1: Validation completed.
[pod/upgrade-helper-cos-state-size-check-l8f9r/validator] ERROR: harvester-node-2: COS_STATE partition is only 8GB, upgrade to Harvester v1.8.x will result in corrupt OS image. Suggest re-installing first to ensure partition is 15GB.
[pod/upgrade-helper-cos-state-size-check-l8f9r/validator] RESULT: harvester-node-2: Validation completed.
[pod/upgrade-helper-cos-state-size-check-xlg5k/validator] ERROR: harvester-node-0: COS_STATE partition is only 8GB, upgrade to Harvester v1.8.x will result in corrupt OS image. Suggest re-installing first to ensure partition is 15GB.
[pod/upgrade-helper-cos-state-size-check-xlg5k/validator] RESULT: harvester-node-0: Validation completed.
COS_STATE-Size Test: Failed

==============================

WARN: There are 1 failing checks: COS_STATE-Size
```

In the above example, all three nodes (harvester-node-0, harvester-node-1, and harvester-node-2) are affected.

On an unaffected system, the `COS_STATE-Size` check will be listed as passing.

## Background on the `COS_STATE` partition, OS image files, and the upgrade process

The `COS_STATE` partition stores two important image files:

- `active.img` - the currently running OS image
- `passive.img` - the previously version of the OS from before the last upgrade (on first install this will be the same as `active.img`)

When you boot a SUSE Virtualization node, by default it will run from `active.img`.  The `passive.img` file is used as a fallback if `active.img` is unable to boot.

During upgrade, SUSE Virtualization calls out to `elemental upgrade`, which will generate a new `active.img` based on the rootfs from the new SUSE Virtualization version, as follows:

1. Create a new `transition.img` (this is the new OS image).
2. Move the current `active.img` to `passive.img`.
3. Move the new `transition.img` to `active.img`.

The `transition.img` created is a sparse file on the `COS_STATE` partition. This image is then loopback mounted, and `elemental` runs `rsync` to copy all the necessary OS files to that image. Because this is a loopback mounted image, from the perspective of `rsync` and `elemental` it appears as like a block device with plenty of free space, so the `rsync` will always succeed. It's the Linux kernel which is responsible for writing the data out to the underlying image file on the `COS_STATE` partition, which happens separately. If that partition runs out of space, you will see I/O errors in the system journal, but the upgrade process has no way of knowing that anything has gone wrong. Here's an example of what the I/O errors look like:

```
Jun 05 07:21:42 harvester-node-0 kernel: critical space allocation error, dev loop2, sector 1454720 op 0x1:(WRITE) flags 0x0 phys_seg 7 prio class 2
Jun 05 07:21:42 harvester-node-0 kernel: EXT4-fs warning (device loop2): ext4_end_bio:354: I/O error 3 writing to inode 121344 starting block 181840)
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181840
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181841
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181842
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181843
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181844
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181845
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181846
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181847
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181848
Jun 05 07:21:42 harvester-node-0 kernel: Buffer I/O error on device loop2, logical block 181849
Jun 05 07:21:42 harvester-node-0 kernel: critical space allocation error, dev loop2, sector 1451736 op 0x1:(WRITE) flags 0x0 phys_seg 7 prio class 2
Jun 05 07:21:42 harvester-node-0 kernel: EXT4-fs warning (device loop2): ext4_end_bio:354: I/O error 3 writing to inode 121342 starting block 181467)
```

## Fixing the problem

### If discovered before upgrade

To future-proof an affected system, you need to make the `COS_STATE` partition equal 15G. There are two ways of doing this:

1. Reinstall the current version of SUSE Virtualization on all affected nodes.
2. Use a tool, we recommend GParted, to resize the `COS_STATE` and `COS_PERSISTENT` partitions and their filesystems in-place.

If your VMs are live-migratable and your cluster has multiple nodes, the above can done with zero downtime for workloads, by reinstalling or repartitioning one node at a time, in a rolling fashion.

If it is not possible to do either of the above, a stop-gap measure is to delete the `passive.img` file from the `COS_STATE` partition prior to upgrade, in order to free up enough space for the upgrade to succeed without corrupting the `active.img` file. This will work for SUSE Virtualization v1.7 to v1.8 upgrades, but is not a permanent solution to the problem.

All the above options are detailed below.

#### Reinstall SUSE Virtualization on all affected nodes

This option requires a SUSE Virtualization cluster with three active control plane nodes, plus ideally at least one more node that can be promoted to a management role temporarily.

For each affected node:

1. Delete that node [using the procedure in the SUSE Virtualization documentation](https://docs.harvesterhci.io/v1.8/host/#deleting-a-node).
   - If this is a control plane node, and there are any non-control plane nodes with either the Default or Management role, SUSE Virtualization will automatically promote one of these other nodes, to ensure that there are three control plane nodes at all times.
2. Reinstall the same version of SUSE Virtualization on the node you just deleted, either by booting the ISO image and working through the installer interactively, or by PXE. You'll need to tell the installer to join the existing cluster. Otherwise, use the same configuration as the node currently has (hostname, network, role, data disk, etc.)
   - Reinstalled nodes may require further post-installation configuration and customisation.
   - Because this is a reinstall, the SUSE Storage default data disk will be wiped. Any additional disks that were added separately via the SUSE Virtualization UI after initial install will not be touched (**TBD additional disks may require extra handling, see testing note below**).
3. Once the node above has successfully joined the cluster, go back to step 1 and repeat for the next node.

Note::

If you have completed the steps above and all nodes in the cluster have the Default or Management role, you can end up with a different set of nodes running the control plane than when you started. However, this is dependent on which nodes are promoted, and when. If you need finer control over this, you can add labels to nodes to define their roles before doing the above: nodes labelled `node-role.harvesterhci.io/management=true` will be consiered for promotion, while nodes labelled `node-role.harvesterhci.io/worker=true` will not.

(**TBD: there's an open issue to document manual role assignment at https://github.com/harvester/harvester/issues/10278**)

> NOTES FROM TSERONG'S TESTING OF THE ABOVE:
>
> - The node deletion procedure in the docs didn't work as adverstised. I suspect this is not exactly a well travelled path...
>   - Replica eviction doesn't always seem to work (the LH GUI may still show 1 replica on the node to be removed)
>   - Running `/opt/rke2/bin/rke2-uninstall.sh` always reports failure, but I barrelled on anyway.
>   - Finally deleting the node from the Harvester GUI never completes. Looks like I'm hitting the problem described in https://github.com/harvester/harvester/issues/10534 and had to add a `machine.cluster.x-k8s.io/exclude-node-draining: "true"` annotation to the affected machine CR to get it unstuck.
>   - The node isn't automatically removed from the Longhorn config (possibly due to me adding the annotation above? or did I just not wait long enough? not sure). This is both good and bad:
>     - Good, because it means any additional disks you've added since initial install will still be there after reinstall
>     - Bad, because you'll get a mismatched diskUUID error for the default disk once the node comes back up after reinstall, then you have to fix that.
>     - If the node were deleted from the longhorn config, which TBH is what you'd _want_ when deleting a node, then additional disks added to that node may need to be re-added after reinstall.
> - Reinstall and rejoin worked fine \o/
> - I don't know what extra caveats there might be for more complex configurations with multiple networks and whatnot.
> - Overall I expect a reinstall to be time consuming and fiddly :-/

#### Use GParted to resize the `COS_STATE` and `COS_PERSISTENT` partitions in-place

The GParted project provides a live CD image which can be used to resize disk partitions in-place. This can be downloaded from https://gparted.org/livecd.php. 

The following procedure will work regardless of how many nodes are in the cluster. For each affected node:

1. Put the node in [maintenance mode](https://docs.harvesterhci.io/v1.8/host/#node-maintenance) to migrate any running VMs away.
   - On clusters with a single control plane node, that node can't be placed in maintenance. In this case, VMs must be migrated off manually.
2. Boot the GParted live image. Once GParted starts, select the system disk, then:
   - Select "Resize/Move" on the `COS_PERSISTENT` partition, and set "Free space preceeding" to 7168 MiB.
   - Select "Resize/Move" on the `COS_STATE` partition, and set "New Size" to 15360 MiB.
   - Apply the changes, wait for it to complete, and reboot.
3. Take the node out of maintenance mode.

This may take some time depending on how much data is on the system disk. Also, if there is a power outage or other catastrophic failure during the repartitioning/resizing process, the system disk may be left in an unusable state, and the node will need to be reinstalled instead.

#### Delete the `passive.img` file before upgrade

If choosing this option, prior to upgrade on each affected node, log in via `ssh` and run the following commands as the root user. Here we check the free disk space on the `COS_STATE` partition, delete the `passive.img` file, then check the free disk space again:

```
# df -h /run/initramfs/cos-state
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda4       7.8G  3.6G  3.8G  49% /run/initramfs/cos-state

# mount -o remount,rw /run/initramfs/cos-state
# rm /run/initramfs/cos-state/cOS/passive.img
# mount -o remount,ro /run/initramfs/cos-state

# df -h /run/initramfs/cos-state
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda4       7.8G  2.0G  5.5G  27% /run/initramfs/cos-state
```

The upgrade process when going from SUSE Virtualization v1.7 to v1.8 uses about 4.3G of disk space on the `COS_STATE` partition while it's generating the new `active.img` file. By deleting the `passive.img` file, 5.5G is made available and the upgrade will succeed.

Note that once the upgrade succeeds, a new `passive.img` file is created. This means that the _next_ time you upgrade, you'll also have to delete the `passive.img` file in advance.

Bear in mind also that as SUSE Virtualization's OS image has increased in size from release to release, that there may come a time in future when deleting `passive.img` will not result in enough free space. We recommend following the future-proof options of either reinstalling or repartitioning the affected nodes in-place.

### If discovered during/after upgrade

The following procedure can be used to recover a node whose `active.img` has become corrupted during upgrade due to the `COS_STATE` partition being too small. Note that this will _not_ fix the underlying problem (the `COS_STATE` partition will still be 8G), it will just get the node back into a running state by re-creating the `active.img` from the v1.8.x rootfs.

1. Reboot into fallback mode (the second grub entry), which will be running SL Micro 6.1 from SUSE Virtualization v1.7.x. NetworkManager will not start at this point, so you have to log in on the console and run `rm /etc/systemd/system/dbus.service` to remove a symlink which is put there by SL Micro 6.2 from SUSE Virtualization v1.8.x. Reboot into fallback mode again, and you'll have networking running correctly again.

2. Log in via `ssh` or on the console and become root, then run the following commands:

   ```
   # mount -o remount,rw /run/initramfs/cos-state
   # cp /run/initramfs/cos-state/cOS/passive.img \
        /run/initramfs/cos-state/cOS/active.img
   # tune2fs -L COS_ACTIVE /run/initramfs/cos-state/cOS/active.img
   # mv /run/initramfs/cos-state/cOS/passive.img /root/
   # mount -o remount,ro /run/initramfs/cos-state
   ```
   This copies the previous (clean) `passive.img` (SL Micro 6.1/Harvester v1.7.x) over the corrupted `active.img` and the label is set correctly for boot.

   This also moves the `passive.img` to the `/root` directory to free up space on the `COS_STATE` partition. This is done rather than just deleting `passive.img` because if anything goes wrong with the following procedure, you will need to undo the previous action and retry.

3. Reboot and select the first entry on the grub screen. This will say it's booting SUSE Virtualization v1.8.x, but will actually boot into the reverted SL Micro 6.1 image from SUSE Virtualization v1.7.x.

4. Copy `rootfs.squashfs` from the SUSE Virtualization v1.8.x ISO to a convenient location, or download it from https://releases.rancher.com/harvester/v1.8.0/harvester-v1.8.0-rootfs-amd64.squashfs or https://releases.rancher.com/harvester/v1.8.1/harvester-v1.8.1-rootfs-amd64.squashfs.

5. As root, run the following commands:
   ```
   # mkdir /tmp/manual-os-upgrade
   # mkdir /tmp/manual-os-upgrade/config
   # mkdir /tmp/manual-os-upgrade/rootfs
   # mount -o loop rootfs.squashfs /tmp/manual-os-upgrade/rootfs
   # cat > /tmp/manual-os-upgrade/config/config.yaml <<EOF
   upgrade:
     system:
       size: 3072
   EOF
   # elemental upgrade \
       --logfile /tmp/manual-os-upgrade/upgrade.log \
       --directory /tmp/manual-os-upgrade/rootfs \
       --config-dir /tmp/manual-os-upgrade/config \
       --debug
   ```
   - This will create a new `active.img` generated based on the SUSE Virtualization v1.8.x rootfs.
   - Be sure to replace the path in the fourth line with the actual path of the rootfs.

6. Assuming there were no errors, unmount the rootfs and reboot:
   ```
   # umount /tmp/manual-os-upgrade/rootfs
   # reboot
   ```

7. The `/etc/systemd/system/dbus.service` symlink that was removed in step 1 will automatically come back when then new v1.8.x active.img boots.
