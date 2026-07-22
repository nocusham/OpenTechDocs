# Sharp MX-3060N Boot Loop and U2-40: mSATA Replacement and Recovery

This document describes a successful repair of a Sharp MX-3060N multifunction copier that repeatedly rebooted during startup and had previously displayed error code `U2-40`.

> **Warning:** The commands below perform raw disk access. A reversed source and destination can destroy data immediately. Verify every device by model, capacity, and serial number before running `ddrescue`. Keep the original drives and master images unchanged until the repair has been fully verified.

## Symptoms

- The copier displayed the Sharp logo after being switched on.
- The display then turned black and the startup sequence began again.
- Fans and motors restarted at the same time, confirming that this was a complete system reboot rather than only a display problem.
- The interval between restarts was much shorter than the normal deep-sleep cycle.
- Error code `U2-40` had been displayed previously.

## Storage configuration

The machine contained two storage devices:

- An Innodisk mSATA 3ME module with a nominal capacity of 16 GB
- A 500 GB Toshiba 2.5-inch SATA hard disk

The mSATA module contained the operating system, boot and recovery data, logs, settings, and several raw/vendor-specific partitions. The hard disk contained additional data partitions.

## Tools used

- [SystemRescue](https://www.system-rescue.org/)
- GNU `ddrescue`
- `ddrescuelog`
- `fdisk`, `lsblk`, `blkid`, and `losetup`
- `e2fsck`
- `smartctl`
- `cmp` and `sha256sum`
- A NAS mounted under Linux for storing the images and mapfiles

## 1. Creating a sector-by-sector backup of the mSATA module

The original mSATA module was connected to the Linux system and identified carefully. In the following examples, `/dev/sdX` is the source device and must be replaced with the correct device name.

```bash
lsblk -o NAME,MODEL,SERIAL,SIZE,LOG-SEC,PHY-SEC,MOUNTPOINTS
```

The image was created directly on the NAS:

```bash
ddrescue -n \
  /dev/sdX \
  /mnt/nas/mx3060-msata.img \
  /mnt/nas/mx3060-msata.map
```

The result was checked with:

```bash
ddrescuelog -t /mnt/nas/mx3060-msata.map
```

Result:

- Image size: `16,013,942,784` bytes
- 100% rescued
- No read errors
- No bad sectors or untried areas

This was an important observation: a flash device can remain completely readable while no longer handling writes reliably. A successful read-only image does not prove that an SSD is healthy.

## 2. Inspecting the mSATA image without modifying it

The image used an MBR/DOS partition table with 14 partition entries. It was attached read-only:

```bash
losetup --find --show --read-only --partscan \
  /mnt/nas/mx3060-msata.img
```

The returned loop device was then inspected:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS /dev/loopN
fdisk -l /mnt/nas/mx3060-msata.img
blkid /dev/loopNp*
```

The partition table was readable and structurally coherent. It included:

- Multiple ext4 partitions
- One extended partition containing logical partitions
- One swap partition
- Two partitions without a standard filesystem signature, presumed to contain vendor-specific raw data

Read-only filesystem checks were performed with `e2fsck -f -n`. The original image was never repaired in place.

```bash
e2fsck -fn /dev/loopNp1
# Repeat only for partitions identified as ext4.
```

The following inconsistencies were found:

- Partition 3: inode, extent, block bitmap, and free-block count inconsistencies
- Partition 5: invalid/corrupt ext4 journal, preventing a complete filesystem check
- Partition 13: block bitmap and free-block count inconsistencies

Other ext4 partitions passed their checks. The errors could have been caused or aggravated by repeated uncontrolled resets. They were preserved in the master image so that all later experiments remained reversible.

## 3. Backing up and checking the 500 GB hard disk

The hard disk was imaged in the same way:

```bash
ddrescue -n \
  /dev/sdX \
  /mnt/nas/TOSHIBA.img \
  /mnt/nas/TOSHIBA.map
```

The backup completed with:

- Image size: `500,107,862,016` bytes
- 100% rescued
- No read errors
- No bad sectors or untried areas

The mapfile and image checksum were recorded:

```bash
ddrescuelog -t /mnt/nas/TOSHIBA.map
sha256sum /mnt/nas/TOSHIBA.img > /mnt/nas/TOSHIBA.img.sha256
```

The HDD image was attached read-only:

```bash
losetup --find --show --read-only --partscan \
  /mnt/nas/TOSHIBA.img
```

All 13 ext4 filesystems completed all five `e2fsck -fn` passes without any reported error. The second partition was an extended partition container and therefore was not checked as a filesystem.

```bash
for p in /dev/loopNp1 /dev/loopNp{3..14}; do
    echo "===== $p ====="
    e2fsck -fn "$p"
done
```

SMART was disabled on this particular HDD. After enabling it, the drive or controller stopped responding while `smartctl` attempted to read the SMART data section. No useful SMART attributes were obtained. This did not invalidate the successful full-disk read or the clean filesystem checks, but the SMART result remained inconclusive.

Based on the complete error-free image and clean filesystems, the HDD was considered much less likely to be the cause of the boot loop.

## 4. Selecting a replacement mSATA SSD

A Transcend MSA230S 64 GB (`TS64GMSA230S`) was selected. It matched the important technical requirements:

- Full-size mSATA form factor
- SATA III interface
- 3.3 V supply
- 512-byte logical sectors
- Capacity greater than the original 16 GB module

The replacement was detected as approximately 59.6 GiB with 512-byte logical and physical sectors.

The larger capacity did not require any partition changes. The original partition table and partition sizes were retained, leaving the remaining space unused. This avoided introducing another variable and preserved the layout expected by the copier.

The original Innodisk module was an industrial mSATA device, whereas the MSA230S is a consumer SSD. The Transcend module proved compatible in this repair, but an industrial mSATA SSD may still be preferable for maximum long-term endurance and power-loss robustness.

## 5. Restoring the original image to the replacement

The destination was verified immediately before writing:

```bash
lsblk -o NAME,MODEL,SERIAL,SIZE,FSTYPE,MOUNTPOINTS /dev/sdY
blockdev --getsize64 /dev/sdY
```

The target must be at least `16,013,942,784` bytes. The image was written to the entire device, not to an individual partition. A new restore mapfile was used so that no previous ddrescue state could cause data to be skipped.

```bash
ddrescue --force \
  /mnt/nas/mx3060-msata.img \
  /dev/sdY \
  /mnt/nas/mx3060-transcend-restore.map

sync
ddrescuelog -t /mnt/nas/mx3060-transcend-restore.map
```

The restore completed with 100% copied and no errors.

The written area was compared byte-for-byte with the source image:

```bash
cmp -n 16013942784 \
  /mnt/nas/mx3060-msata.img \
  /dev/sdY

echo $?
```

`cmp` returned `0`, confirming an exact match. The partition table was re-read and checked:

```bash
partprobe /dev/sdY
lsblk -o NAME,SIZE,FSTYPE,UUID /dev/sdY
```

All expected partitions and filesystem types were present.

## 6. First boot test

For the first test:

- The new mSATA contained an unmodified, bit-for-bit copy of the original image.
- The existing Toshiba HDD remained installed and unchanged.
- No partitions were resized.
- No filesystem repairs were performed on the replacement before booting.

Changing only the mSATA device made the test diagnostically useful: if the machine booted, the old mSATA hardware was the likely cause; if the loop remained, the next step would have been filesystem repair only on the disposable replacement copy.

## Result

With the cloned Transcend MSA230S installed, the Sharp MX-3060N booted normally and all tested functions worked normally. The reboot loop and `U2-40` condition did not return during the test.

No full firmware recovery was required, and the original 500 GB hard disk was retained.

## Conclusion

The original Innodisk mSATA module was most likely failing in a way that affected reliable writes while leaving reads intact. This explains why:

- GNU ddrescue could create a complete image with zero read errors.
- The cloned image worked when written to a new mSATA SSD.
- Replacing only the mSATA module resolved the full-system reboot loop.

This case demonstrates that zero read errors do not rule out SSD failure. In embedded equipment, a flash module that has become read-only or unreliable during writes can cause boot loops, watchdog resets, journal corruption, or persistent storage-related error codes.

## Recommended follow-up

- Keep the original mSATA and HDD images unchanged.
- Store the images, mapfiles, and checksums in at least two locations.
- Do not enlarge the cloned partitions unless there is a documented service requirement.
- Use the copier's normal shutdown procedure and wait for shutdown to complete before operating the main power switch.
- After several successful boots, optionally create a new image of the known-working replacement. Only the original image length is required for restoring the preserved layout:

```bash
ddrescue -n --size=16013942784 \
  /dev/sdY \
  /mnt/nas/mx3060-msata-known-good.img \
  /mnt/nas/mx3060-msata-known-good.map
```

Always identify `/dev/sdY` again before running this command; Linux device names can change between boots.

## Privacy note

Device serial numbers, filesystem UUIDs, checksums, network identifiers, and personal configuration data have intentionally been omitted from this report.
